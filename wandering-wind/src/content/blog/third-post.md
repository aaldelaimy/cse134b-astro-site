---
title: "Building a Custom Unix-Like Shell in C"
description: "Deep dive into system programming: implementing a functional shell with job control, piping, and signal handling"
pubDate: "Mar 10 2024"
---

# Building a Custom Unix-Like Shell in C

System-level programming is fundamental to understanding how operating systems work. In this post, I'll share my experience building a custom Unix-like shell in C, exploring the intricacies of process management, job control, and low-level system calls.

## Project Motivation

Understanding how shells work at a fundamental level requires diving deep into system programming concepts. My goal was to create a functional shell that could handle:

- Command parsing and execution
- Process management and job control
- Input/output redirection
- Piping between processes
- Background job execution
- Signal handling

## Core Architecture

The shell follows a simple but powerful architecture:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Input Parser  │───►│  Command Executor│───►│  Job Controller │
│                 │    │                 │    │                 │
│ • Tokenization  │    │ • Process Creation│   │ • Background Jobs│
│ • Syntax Check  │    │ • Redirection   │    │ • Signal Handling│
│ • Command Build │    │ • Piping        │    │ • Job Status     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Command Parsing Implementation

The first challenge was implementing robust command parsing:

```c
typedef struct {
    char **args;
    int argc;
    char *input_file;
    char *output_file;
    int background;
    int pipe_to_next;
} command_t;

command_t* parse_command(char *line) {
    command_t *cmd = malloc(sizeof(command_t));
    cmd->args = NULL;
    cmd->argc = 0;
    cmd->input_file = NULL;
    cmd->output_file = NULL;
    cmd->background = 0;
    cmd->pipe_to_next = 0;
    
    // Tokenize the input line
    char *token = strtok(line, " \t\n");
    while (token != NULL) {
        if (strcmp(token, ">") == 0) {
            // Output redirection
            token = strtok(NULL, " \t\n");
            cmd->output_file = strdup(token);
        } else if (strcmp(token, "<") == 0) {
            // Input redirection
            token = strtok(NULL, " \t\n");
            cmd->input_file = strdup(token);
        } else if (strcmp(token, "&") == 0) {
            // Background execution
            cmd->background = 1;
        } else if (strcmp(token, "|") == 0) {
            // Pipe to next command
            cmd->pipe_to_next = 1;
        } else {
            // Regular argument
            cmd->args = realloc(cmd->args, (cmd->argc + 1) * sizeof(char*));
            cmd->args[cmd->argc++] = strdup(token);
        }
        token = strtok(NULL, " \t\n");
    }
    
    return cmd;
}
```

## Process Execution

The core of the shell is process creation and execution:

```c
int execute_command(command_t *cmd) {
    pid_t pid = fork();
    
    if (pid == 0) {
        // Child process
        setup_redirections(cmd);
        
        if (execvp(cmd->args[0], cmd->args) == -1) {
            perror("execvp failed");
            exit(EXIT_FAILURE);
        }
    } else if (pid > 0) {
        // Parent process
        if (!cmd->background) {
            // Wait for foreground process
            int status;
            waitpid(pid, &status, 0);
        } else {
            // Add to background jobs
            add_background_job(pid, cmd);
            printf("[%d] %d\n", get_next_job_id(), pid);
        }
    } else {
        perror("fork failed");
        return -1;
    }
    
    return 0;
}
```

## Input/Output Redirection

Implementing redirection required careful file descriptor management:

```c
void setup_redirections(command_t *cmd) {
    // Input redirection
    if (cmd->input_file) {
        int fd = open(cmd->input_file, O_RDONLY);
        if (fd == -1) {
            perror("open input file failed");
            exit(EXIT_FAILURE);
        }
        dup2(fd, STDIN_FILENO);
        close(fd);
    }
    
    // Output redirection
    if (cmd->output_file) {
        int fd = open(cmd->output_file, O_WRONLY | O_CREAT | O_TRUNC, 0644);
        if (fd == -1) {
            perror("open output file failed");
            exit(EXIT_FAILURE);
        }
        dup2(fd, STDOUT_FILENO);
        close(fd);
    }
}
```

## Piping Implementation

Piping between processes was one of the most challenging aspects:

```c
int execute_pipeline(command_t **commands, int num_commands) {
    int pipes[num_commands - 1][2];
    
    // Create pipes
    for (int i = 0; i < num_commands - 1; i++) {
        if (pipe(pipes[i]) == -1) {
            perror("pipe failed");
            return -1;
        }
    }
    
    // Execute commands
    for (int i = 0; i < num_commands; i++) {
        pid_t pid = fork();
        
        if (pid == 0) {
            // Child process
            if (i > 0) {
                // Read from previous pipe
                dup2(pipes[i-1][0], STDIN_FILENO);
            }
            
            if (i < num_commands - 1) {
                // Write to next pipe
                dup2(pipes[i][1], STDOUT_FILENO);
            }
            
            // Close all pipe file descriptors
            for (int j = 0; j < num_commands - 1; j++) {
                close(pipes[j][0]);
                close(pipes[j][1]);
            }
            
            // Execute command
            if (execvp(commands[i]->args[0], commands[i]->args) == -1) {
                perror("execvp failed");
                exit(EXIT_FAILURE);
            }
        }
    }
    
    // Parent process: close all pipes and wait for children
    for (int i = 0; i < num_commands - 1; i++) {
        close(pipes[i][0]);
        close(pipes[i][1]);
    }
    
    for (int i = 0; i < num_commands; i++) {
        wait(NULL);
    }
    
    return 0;
}
```

## Job Control

Background job management required careful process tracking:

```c
typedef struct {
    pid_t pid;
    int job_id;
    char *command;
    int status; // 0: running, 1: stopped, 2: completed
} job_t;

job_t *background_jobs[MAX_JOBS];
int num_jobs = 0;

void add_background_job(pid_t pid, command_t *cmd) {
    if (num_jobs >= MAX_JOBS) {
        fprintf(stderr, "Too many background jobs\n");
        return;
    }
    
    job_t *job = malloc(sizeof(job_t));
    job->pid = pid;
    job->job_id = num_jobs + 1;
    job->command = strdup(cmd->args[0]);
    job->status = 0;
    
    background_jobs[num_jobs++] = job;
}

void check_background_jobs() {
    for (int i = 0; i < num_jobs; i++) {
        if (background_jobs[i] && background_jobs[i]->status == 0) {
            int status;
            pid_t result = waitpid(background_jobs[i]->pid, &status, WNOHANG);
            
            if (result > 0) {
                if (WIFEXITED(status)) {
                    printf("[%d] Done %s\n", background_jobs[i]->job_id, 
                           background_jobs[i]->command);
                    background_jobs[i]->status = 2;
                } else if (WIFSIGNALED(status)) {
                    printf("[%d] Terminated %s\n", background_jobs[i]->job_id,
                           background_jobs[i]->command);
                    background_jobs[i]->status = 2;
                }
            }
        }
    }
}
```

## Signal Handling

Proper signal handling is crucial for a functional shell:

```c
void signal_handler(int sig) {
    if (sig == SIGINT) {
        printf("\n");
        // Don't exit, just print new prompt
    } else if (sig == SIGTSTP) {
        // Handle Ctrl+Z (suspend)
        printf("\n");
    }
}

void setup_signal_handlers() {
    signal(SIGINT, signal_handler);
    signal(SIGTSTP, signal_handler);
    
    // Ignore SIGQUIT for background processes
    signal(SIGQUIT, SIG_IGN);
}
```

## Built-in Commands

The shell includes several built-in commands:

```c
int execute_builtin(command_t *cmd) {
    if (strcmp(cmd->args[0], "cd") == 0) {
        if (cmd->argc < 2) {
            chdir(getenv("HOME"));
        } else {
            if (chdir(cmd->args[1]) != 0) {
                perror("cd failed");
            }
        }
        return 1;
    } else if (strcmp(cmd->args[0], "jobs") == 0) {
        for (int i = 0; i < num_jobs; i++) {
            if (background_jobs[i] && background_jobs[i]->status == 0) {
                printf("[%d] %s\n", background_jobs[i]->job_id,
                       background_jobs[i]->command);
            }
        }
        return 1;
    } else if (strcmp(cmd->args[0], "exit") == 0) {
        exit(EXIT_SUCCESS);
    }
    
    return 0; // Not a built-in command
}
```

## Main Shell Loop

The main execution loop ties everything together:

```c
void shell_loop() {
    char *line;
    char prompt[256];
    
    while (1) {
        // Check for completed background jobs
        check_background_jobs();
        
        // Display prompt
        snprintf(prompt, sizeof(prompt), "%s$ ", getcwd(NULL, 0));
        printf("%s", prompt);
        
        // Read input
        line = readline();
        if (!line) break;
        
        if (strlen(line) == 0) continue;
        
        // Parse and execute command
        command_t *cmd = parse_command(line);
        if (cmd->argc > 0) {
            if (!execute_builtin(cmd)) {
                execute_command(cmd);
            }
        }
        
        // Clean up
        free_command(cmd);
        free(line);
    }
}
```

## Performance Optimizations

Several optimizations were implemented for better performance:

### 1. Memory Management

```c
void free_command(command_t *cmd) {
    if (!cmd) return;
    
    for (int i = 0; i < cmd->argc; i++) {
        free(cmd->args[i]);
    }
    free(cmd->args);
    free(cmd->input_file);
    free(cmd->output_file);
    free(cmd);
}
```

### 2. Command History

```c
#define MAX_HISTORY 1000
char *command_history[MAX_HISTORY];
int history_count = 0;

void add_to_history(const char *command) {
    if (history_count >= MAX_HISTORY) {
        free(command_history[0]);
        memmove(command_history, command_history + 1, 
                (MAX_HISTORY - 1) * sizeof(char*));
        history_count--;
    }
    
    command_history[history_count++] = strdup(command);
}
```

## Testing and Validation

The shell was thoroughly tested with various scenarios:

- **Basic commands**: `ls`, `cat`, `grep`, `wc`
- **Redirection**: `cat < input.txt > output.txt`
- **Piping**: `ls | grep .txt | wc -l`
- **Background jobs**: `sleep 10 &`
- **Job control**: `jobs`, `fg`, `bg`
- **Signal handling**: Ctrl+C, Ctrl+Z

## Key Learnings

This project reinforced several fundamental concepts:

1. **Process management**: Understanding fork(), exec(), and wait()
2. **File descriptors**: Managing stdin, stdout, and stderr
3. **Signal handling**: Proper signal management for robust operation
4. **Memory management**: Careful allocation and deallocation in C
5. **System calls**: Direct interaction with the operating system

## Challenges Encountered

### 1. Process Synchronization

Managing multiple processes and their interactions was complex:
- **Zombie processes**: Proper cleanup of terminated processes
- **Signal propagation**: Ensuring signals reach the correct processes
- **Resource management**: Preventing file descriptor leaks

### 2. Memory Management

C requires careful memory management:
- **Memory leaks**: Ensuring all allocated memory is freed
- **Buffer overflows**: Proper bounds checking
- **String handling**: Safe string operations

### 3. Error Handling

Robust error handling is essential:
- **System call failures**: Handling failed fork(), exec(), etc.
- **Invalid commands**: Graceful handling of non-existent programs
- **Permission errors**: Proper error messages for access issues

## Future Enhancements

The shell could be extended with:

- **Tab completion**: Command and filename completion
- **Aliases**: User-defined command shortcuts
- **Environment variables**: Full environment variable support
- **Scripting**: Support for shell scripts
- **Advanced job control**: More sophisticated job management

## Conclusion

Building a custom shell was an excellent exercise in system programming. The project provided deep insights into how operating systems work at the process level and how shells interact with the kernel.

The combination of process management, signal handling, and low-level system calls created a functional tool that demonstrates the power of C for system-level programming. The skills gained from this project have been invaluable for understanding other system-level concepts and have influenced my approach to other low-level programming tasks.

*This project showcases my expertise in C programming, system programming, process management, and low-level Unix system calls. The complete implementation is available on my GitHub repository.*
