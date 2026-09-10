#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

void display_parent_info() {
    printf("[Parent Info] Current PID: %d, Parent PID (PPID): %d\n", getpid(), getppid());
}

void display_child_info() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return;
    } else if (pid == 0) {
        // Inside child process
        printf("[Child Info] Child PID: %d, Parent PID (PPID): %d\n", getpid(), getppid());
        exit(0); // Terminate child so it does not loop back to menu
    } else {
        // Parent waits so output doesn't interleave
        wait(NULL);
    }
}

void display_process_hierarchy() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return;
    } else if (pid == 0) {
        printf("[Hierarchy - Child]  PID: %d, PPID: %d\n", getpid(), getppid());
        exit(0);
    } else {
        printf("[Hierarchy - Parent] PID: %d, Child PID: %d, Grandparent PID (PPID): %d\n", 
               getpid(), pid, getppid());
        wait(NULL);
    }
}

void make_parent_sleep() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return;
    } else if (pid == 0) {
        printf("[Child] Running immediately. PID: %d\n", getpid());
        exit(0);
    } else {
        printf("[Parent] Going to sleep for 3 seconds...\n");
        sleep(3);
        printf("[Parent] Woke up.\n");
        wait(NULL);
    }
}

void make_child_sleep() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return;
    } else if (pid == 0) {
        printf("[Child] Going to sleep for 3 seconds...\n");
        sleep(3);
        printf("[Child] Woke up. Exiting.\n");
        exit(0);
    } else {
        printf("[Parent] Waiting for sleeping child...\n");
        wait(NULL);
        printf("[Parent] Child finished sleeping.\n");
    }
}

void terminate_child() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return;
    } else if (pid == 0) {
        printf("[Child] Terminating explicitly with exit code 42.\n");
        exit(42);
    } else {
        int status;
        wait(&status);
        if (WIFEXITED(status)) {
            printf("[Parent] Child terminated with exit status: %d\n", WEXITSTATUS(status));
        }
    }
}

void terminate_parent() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return;
    } else if (pid == 0) {
        printf("[Child] Started. Sleeping 2 seconds so parent can terminate first...\n");
        sleep(2);
        // After parent exits, this child is adopted by init / systemd (PPID becomes 1 or user systemd PID)
        printf("[Child] Orphan check: New PPID: %d. Terminating.\n", getppid());
        exit(0);
    } else {
        printf("[Parent] Terminating immediately. Child (PID: %d) will become an orphan.\n", pid);
        exit(0); // Terminates the main program loop as well
    }
}

void wait_child_wait() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return;
    } else if (pid == 0) {
        printf("[Child] Running some work...\n");
        sleep(1);
        printf("[Child] Done.\n");
        exit(0);
    } else {
        printf("[Parent] Blocked on wait()...\n");
        pid_t terminated_pid = wait(NULL);
        printf("[Parent] wait() caught child process with PID: %d\n", terminated_pid);
    }
}

void wait_child_waitpid() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return;
    } else if (pid == 0) {
        printf("[Child] Running background work...\n");
        sleep(2);
        printf("[Child] Done.\n");
        exit(0);
    } else {
        int status;
        printf("[Parent] Waiting specifically for child PID: %d via waitpid()...\n", pid);
        waitpid(pid, &status, 0); // 0 means wait until specified child terminates
        if (WIFEXITED(status)) {
            printf("[Parent] waitpid() confirmed exit of child %d.\n", pid);
        }
    }
}

void launch_new_program() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return;
    } else if (pid == 0) {
        printf("[Child] Replacing process image with 'ls' using execlp()...\n");
        execlp("ls", "ls", NULL);
        // If execlp returns, it failed
        perror("execlp failed");
        exit(1);
    } else {
        wait(NULL);
        printf("[Parent] 'ls' execution complete.\n");
    }
}

void launch_program_with_args() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return;
    } else if (pid == 0) {
        printf("[Child] Executing 'ls -l -a' using execvp()...\n");
        char *args[] = {"ls", "-l", "-a", NULL};
        execvp(args[0], args);
        perror("execvp failed");
        exit(1);
    } else {
        wait(NULL);
        printf("[Parent] Command with arguments complete.\n");
    }
}

int main() {
    int choice;

    while (1) {
        printf("\n=========================================\n");
        printf("1. Display Parent Information\n");
        printf("2. Display Child Information\n");
        printf("3. Display Process Hierarchy\n");
        printf("4. Make Parent Sleep\n");
        printf("5. Make Child Sleep\n");
        printf("6. Terminate Child\n");
        printf("7. Terminate Parent (Orphans Child)\n");
        printf("8. Wait for Child using wait()\n");
        printf("9. Wait for Child using waitpid()\n");
        printf("10. Launch New Program\n");
        printf("11. Launch Program with Arguments\n");
        printf("0. Exit\n");
        printf("=========================================\n");
        printf("Enter your choice: ");

        if (scanf("%d", &choice) != 1) {
            printf("Invalid input. Exiting.\n");
            break;
        }

        switch (choice) {
            case 1:  display_parent_info(); break;
            case 2:  display_child_info(); break;
            case 3:  display_process_hierarchy(); break;
            case 4:  make_parent_sleep(); break;
            case 5:  make_child_sleep(); break;
            case 6:  terminate_child(); break;
            case 7:  terminate_parent(); break;
            case 8:  wait_child_wait(); break;
            case 9:  wait_child_waitpid(); break;
            case 10: launch_new_program(); break;
            case 11: launch_program_with_args(); break;
            case 0:
                printf("Exiting program.\n");
                exit(0);
            default:
                printf("Invalid choice! Please select 0-11.\n");
        }
    }

    return 0;
}