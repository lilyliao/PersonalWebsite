# Building a Custom Shell from Scratch

In the world of computing, the shell is an indispensable tool for developers, system administrators, and power users alike. It's the interface that allows us to converse with the operating system, to tell it what to do, and to automate complex tasks. But what if the shells you've used don't do exactly what you want? What if you need a feature that doesn't exist, or you want to understand at a deeper level how your computer executes your commands? The answer is to write your own shell.

Before we dive into the how, let's clarify what a shell is. A shell is a command-line interpreter that provides a user interface for the Unix/Linux operating system services. Popular shells include Bash, Zsh, and Fish, each with their own set of features and capabilities.

### Why Write Your Own Shell?

Writing your own shell can be an enlightening experience. It can help you:

* Understand how the operating system works at a deeper level.
* Learn how commands are executed and how processes are managed.
* Customize your working environment to fit your exact needs.
* Improve your programming skills, particularly in systems programming.

### Getting Started

To write a shell, you need a good grasp of programming. C is the traditional language for this task due to its close relationship with Unix, but you can also use more modern languages like Rust or Go if you're more comfortable with them.

Here are the basic steps you'll follow to create your own shell:

1. **Initialize the Shell**: Start by setting up a loop that will keep your shell running and waiting for user input.
2. **Parse Input**: Read the command-line input and parse it into commands and arguments.
3. **Execute Commands**: Use system calls to execute the commands you've parsed.
4. **Manage Processes**: Handle the creation of new processes and ensure they execute correctly, including foreground and background processes.
5. **Implement Built-ins**: Add built-in commands like `cd`, `help`, or `exit`.
6. **Handle Signals**: Deal with signals like `SIGINT` and `SIGCHLD`.
7. **Expand Features**: Add scripting capabilities, command history, tab completion, and more.

#### Step 1: Initialize Your Shell

The initialization of your shell involves setting up a read-eval-print loop (REPL). Here's a more fleshed-out version in C:

```
c
```

```c
#include <stdio.h>
#include <stdbool.h>

void read_input(char* input) {
    fgets(input, 1024, stdin);
}

int main() {
    char input[1024];
    while (true) {
        printf("myshell> ");
        fflush(stdout); // Make sure "myshell> " is printed immediately
        read_input(input);
        // Process and execute the input here
    }
    return 0;
}
```

#### Step 2: Parse the Input

Parsing the input involves tokenizing the string you've received. Here's a simple tokenizer using `strtok` in C:

```
c
```

```c
#include <string.h>

void parse_input(char* input, char** argv) {
    char* token;
    token = strtok(input, " \n");
    int i = 0;
    while (token != NULL) {
        argv[i] = token;
        token = strtok(NULL, " \n");
        i++;
    }
    argv[i] = NULL; // Null-terminate the array of arguments
}
```

#### Step 3: Execute Commands

To execute commands, you'll need to fork the process and use `execvp` to run the command:

```
c
```

```c
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

void execute_command(char** argv) {
    pid_t pid = fork();
    if (pid == -1) {
        // Error handling
        perror("fork");
    } else if (pid == 0) {
        // Child process
        if (execvp(argv[0], argv) == -1) {
            // Error handling
            perror("execvp");
        }
        exit(EXIT_FAILURE); // Exit if execvp fails
    } else {
        // Parent process
        int status;
        waitpid(pid, &status, 0); // Wait for the child process to finish
    }
}
```

#### Step 4: Process Management

Managing processes involves handling foreground and background execution. Here's a snippet for foreground process management:

```
c
```

```c
void execute_command(char** argv, bool run_in_background) {
    pid_t pid = fork();
    if (pid == -1) {
        // Error handling
        perror("fork");
    } else if (pid == 0) {
        // Child process
        if (execvp(argv[0], argv) == -1) {
            // Error handling
            perror("execvp");
        }
        exit(EXIT_FAILURE);
    } else {
        // Parent process
        if (!run_in_background) {
            int status;
            waitpid(pid, &status, 0);
        } else {
            printf("Process running in background with PID: %d\n", pid);
        }
    }
}
```

#### Step 5: Implementing Built-ins

For built-in commands, you'll need to check the command name and perform the action directly. Here's an example for `cd`:

```
c
```

```c
#include <unistd.h>

int change_directory(char* path) {
    if (chdir(path) == -1) {
        // Error handling
        perror("chdir");
        return -1;
    }
    return 0;
}

// In your command execution logic
if (strcmp(argv[0], "cd") == 0) {
    change_directory(argv[1]);
}
```

#### Step 6: Handle Signals

Handling signals involves setting up signal handlers. Here's a simple example for `SIGINT`:

```
c
```

```c
#include <signal.h>

void sigint_handler(int sig) {
    write(1, "\nmyshell> ", 10);
}

int main() {
    signal(SIGINT, sigint_handler);
    // ... rest of your shell loop
}
```

#### Step 7: Expand Features

As an example of expanding features, let's add a simple history feature using a linked list:

```
```

```c
#include <stdlib.h>

typedef struct CommandHistory {
    char* command;
    struct CommandHistory* next;
} CommandHistory;

CommandHistory* add_history(CommandHistory* history, char* command) {
    CommandHistory* new_entry = malloc(sizeof(CommandHistory));
    new_entry->command = strdup(command); // Duplicate the command string
    new_entry->next = history;
    return new_entry;
}

// Usage within your main loop
history = add_history(history, input);
```
