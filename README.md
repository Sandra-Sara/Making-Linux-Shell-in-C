# Making-Linux-Shell-in-C

Step 1: Understand What a Shell Does
Before coding, it’s important to know the responsibilities of a shell:

1.Display a prompt to the user.
2.Accept user input (commands).
3.Store command history.
4.Parse the command into tokens.
5.Check for built-in commands (e.g., cd, exit, help).
6.Handle special characters like pipes (|).
7.Execute system commands using fork() and execvp().
8.Wait for child processes to complete.
9.Repeat the process in a loop.

Step 2: Install Required Library
The program uses the GNU readline library for command history and better input handling.
@Bash
sudo apt update
sudo apt install libreadline-dev
Verify installation:
@Bash
dpkg -l | grep readline

Step 3: Create the Source File
Create a new C file for your shell:
shell.c
// C Program to design a shell in Linux
#include<stdio.h>
#include<string.h>
#include<stdlib.h>
#include<unistd.h>
#include<sys/types.h>
#include<sys/wait.h>
#include<readline/readline.h>
#include<readline/history.h>

#define MAXCOM 1000 // max number of letters to be supported
#define MAXLIST 100 // max number of commands to be supported

// Clearing the shell using escape sequences
#define clear() printf("\033[H\033[J")

// Greeting shell during startup
void init_shell()
{
    clear();
    printf("\n\n\n\n******************"
        "************************");
    printf("\n\n\n\t****MY SHELL****");
    printf("\n\n\t-USE AT YOUR OWN RISK-");
    printf("\n\n\n\n*******************"
        "***********************");
    char* username = getenv("USER");
    printf("\n\n\nUSER is: @%s", username);
    printf("\n");
    sleep(1);
    clear();
}

// Function to take input
int takeInput(char* str)
{
    char* buf;

    buf = readline("\n>>> ");
    if (strlen(buf) != 0) {
        add_history(buf);
        strcpy(str, buf);
        return 0;
    } else {
        return 1;
    }
}

// Function to print Current Directory.
void printDir()
{
    char cwd[1024];
    getcwd(cwd, sizeof(cwd));
    printf("\nDir: %s", cwd);
}

// Function where the system command is executed
void execArgs(char** parsed)
{
    // Forking a child
    pid_t pid = fork(); 

    if (pid == -1) {
        printf("\nFailed forking child..");
        return;
    } else if (pid == 0) {
        if (execvp(parsed[0], parsed) < 0) {
            printf("\nCould not execute command..");
        }
        exit(0);
    } else {
        // waiting for child to terminate
        wait(NULL); 
        return;
    }
}

// Function where the piped system commands is executed
void execArgsPiped(char** parsed, char** parsedpipe)
{
    // 0 is read end, 1 is write end
    int pipefd[2]; 
    pid_t p1, p2;

    if (pipe(pipefd) < 0) {
        printf("\nPipe could not be initialized");
        return;
    }
    p1 = fork();
    if (p1 < 0) {
        printf("\nCould not fork");
        return;
    }

    if (p1 == 0) {
        // Child 1 executing..
        // It only needs to write at the write end
        close(pipefd[0]);
        dup2(pipefd[1], STDOUT_FILENO);
        close(pipefd[1]);

        if (execvp(parsed[0], parsed) < 0) {
            printf("\nCould not execute command 1..");
            exit(0);
        }
    } else {
        // Parent executing
        p2 = fork();

        if (p2 < 0) {
            printf("\nCould not fork");
            return;
        }

        // Child 2 executing..
        // It only needs to read at the read end
        if (p2 == 0) {
            close(pipefd[1]);
            dup2(pipefd[0], STDIN_FILENO);
            close(pipefd[0]);
            if (execvp(parsedpipe[0], parsedpipe) < 0) {
                printf("\nCould not execute command 2..");
                exit(0);
            }
        } else {
            // parent executing, waiting for two children
            wait(NULL);
            wait(NULL);
        }
    }
}

// Help command builtin
void openHelp()
{
    puts("\n***WELCOME TO MY SHELL HELP***"
        "\nCopyright @ Suprotik Dey"
        "\n-Use the shell at your own risk..."
        "\nList of Commands supported:"
        "\n>cd"
        "\n>ls"
        "\n>exit"
        "\n>all other general commands available in UNIX shell"
        "\n>pipe handling"
        "\n>improper space handling");

    return;
}

// Function to execute builtin commands
int ownCmdHandler(char** parsed)
{
    int NoOfOwnCmds = 4, i, switchOwnArg = 0;
    char* ListOfOwnCmds[NoOfOwnCmds];
    char* username;

    ListOfOwnCmds[0] = "exit";
    ListOfOwnCmds[1] = "cd";
    ListOfOwnCmds[2] = "help";
    ListOfOwnCmds[3] = "hello";

    for (i = 0; i < NoOfOwnCmds; i++) {
        if (strcmp(parsed[0], ListOfOwnCmds[i]) == 0) {
            switchOwnArg = i + 1;
            break;
        }
    }

    switch (switchOwnArg) {
    case 1:
        printf("\nGoodbye\n");
        exit(0);
    case 2:
        chdir(parsed[1]);
        return 1;
    case 3:
        openHelp();
        return 1;
    case 4:
        username = getenv("USER");
        printf("\nHello %s.\nMind that this is "
            "not a place to play around."
            "\nUse help to know more..\n",
            username);
        return 1;
    default:
        break;
    }

    return 0;
}

// function for finding pipe
int parsePipe(char* str, char** strpiped)
{
    int i;
    for (i = 0; i < 2; i++) {
        strpiped[i] = strsep(&str, "|");
        if (strpiped[i] == NULL)
            break;
    }

    if (strpiped[1] == NULL)
        return 0; // returns zero if no pipe is found.
    else {
        return 1;
    }
}

// function for parsing command words
void parseSpace(char* str, char** parsed)
{
    int i;

    for (i = 0; i < MAXLIST; i++) {
        parsed[i] = strsep(&str, " ");

        if (parsed[i] == NULL)
            break;
        if (strlen(parsed[i]) == 0)
            i--;
    }
}

int processString(char* str, char** parsed, char** parsedpipe)
{

    char* strpiped[2];
    int piped = 0;

    piped = parsePipe(str, strpiped);

    if (piped) {
        parseSpace(strpiped[0], parsed);
        parseSpace(strpiped[1], parsedpipe);

    } else {

        parseSpace(str, parsed);
    }

    if (ownCmdHandler(parsed))
        return 0;
    else
        return 1 + piped;
}

int main()
{
    char inputString[MAXCOM], *parsedArgs[MAXLIST];
    char* parsedArgsPiped[MAXLIST];
    int execFlag = 0;
    init_shell();

    while (1) {
        // print shell line
        printDir();
        // take input
        if (takeInput(inputString))
            continue;
        // process
        execFlag = processString(inputString,
        parsedArgs, parsedArgsPiped);
        // execflag returns zero if there is no command
        // or it is a builtin command,
        // 1 if it is a simple command
        // 2 if it is including a pipe.

        // execute
        if (execFlag == 1)
            execArgs(parsedArgs);

        if (execFlag == 2)
            execArgsPiped(parsedArgs, parsedArgsPiped);
    }
    return 0;
}
Paste the complete code you provided into this file, then save and exit:
Press CTRL + O → Enter to save.
Press CTRL + X to exit.

Step 4: Compile the Program
Compile the shell using gcc and link the readline library:
@Bash
gcc shell.c -o myshell -lreadline
If compilation is successful, an executable named myshell will be created.


Step 5: Run the Shell
Execute your custom shell:
@Bash
./myshell
You should see a welcome message and a prompt displaying the current directory:
Dir: /home/username
>>>

Step 6: Test Basic Commands
Try running some common Linux commands:
@Bash
>>> ls
>>> pwd
>>> date
>>> whoami
These commands are executed using fork() and execvp().


Step 7: Test Built-in Commands

Your shell supports several built-in commands:

Command	Description
cd <directory>	Change the current directory
help	Display help information
hello	Display a greeting
exit	Exit the shell

Example:
@Bash
>>> cd ..
>>> help
>>> hello
>>> exit

Step 8: Test Pipe Functionality

The shell supports a single pipe (|), allowing the output of one command to be used as input to another.

Example:

>>> ls | wc -l
>>> cat file.txt | grep hello
How It Works Internally
pipe(pipefd) creates two file descriptors:
pipefd[0]: Read end.
pipefd[1]: Write end.
Two child processes are created using fork().
dup2() redirects:
First child: STDOUT → write end of pipe.
Second child: STDIN → read end of pipe.
execvp() runs the respective commands.
The parent waits for both children using wait().

Step 9: Understand the Key Functions
1. init_shell()
Clears the screen and displays a welcome message.
Retrieves the username using getenv("USER").
2. takeInput()
Uses readline() to accept input.
Stores commands in history with add_history().
3. printDir()
Displays the current working directory using getcwd().
4. parseSpace()
Splits the command into tokens based on spaces using strsep().
5. parsePipe()
Detects and separates commands connected by a pipe (|).
6. ownCmdHandler()
Handles built-in commands like cd, help, exit, and hello.
7. execArgs()
Executes non-piped system commands using fork() and execvp().
8. execArgsPiped()
Handles execution of two commands connected by a pipe.
9. processString()
Coordinates parsing and determines whether the command is built-in, simple, or piped.

Step 10: Program Execution Flow
Start
  ↓
Initialize Shell
  ↓
Display Current Directory
  ↓
Read User Input
  ↓
Parse Command
  ↓
Check Built-in Command?
  ├─ Yes → Execute Built-in
  └─ No
       ↓
   Pipe Present?
       ├─ Yes → Execute Piped Commands
       └─ No  → Execute Simple Command
  ↓
Wait for Child Processes
  ↓
Repeat Loop


