# bank_management_system_project
this is a banking project it provide the facility of:1.create account  2.credit & debit money 3.transfer money 4.view history and check balance.
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <time.h>

// Function Declarations
void create_account();
void deposit_money();
void withdraw_money();
void check_balance();
void delete_account();
void transfer_money();
void view_transaction_history();

// Global File Constants
const char *ACCOUNT_FILE = "account.dat";
const char *TRANSACTION_FILE = "transactions.dat";

// Structures
typedef struct {
    char name[50];
    int acc_no;
    float balance;
} account;

typedef struct {
    int acc_no;
    char type[15]; // "Deposit", "Withdrawal", "Transfer Out", "Transfer In"
    float amount;
    char date_time[30];
} transaction;

int main() {
    int choice;
    while (1) {
        printf("\n**** Banking Management System ****\n");
        printf("1. Create Account\n");
        printf("2. Deposit Money\n");
        printf("3. Withdraw Money\n");
        printf("4. Transfer Money\n");
        printf("5. Check Balance\n");
        printf("6. Delete Account\n");
        printf("7. View Transaction History\n");
        printf("8. Exit\n");
        printf("* Enter your choice: ");
        
        if (scanf("%d", &choice) != 1) {
            printf("Invalid input. Please enter a valid menu number.\n");
            while (getchar() != '\n'); // Clear buffer
            continue;
        }

        switch (choice) {
            case 1:
                create_account();
                break;
            case 2:
                deposit_money();
                break;
            case 3:
                withdraw_money();
                break;
            case 4:
                transfer_money();
                break;
            case 5:
                check_balance();
                break;
            case 6:
                delete_account();
                break;
            case 7:
                view_transaction_history();
                break;
            case 8:
                printf("\nClosing the bank. Thanks for your visit!\n");
                exit(0);
            default:
                printf("Invalid choice! Please select between 1 and 8.\n");
                break;
        }
    }  
    return 0;
}

// Helper function to automatically log transactions with current timestamps
void log_transaction(int acc_no, const char *type, float amount) {
    FILE *t_file = fopen(TRANSACTION_FILE, "ab");
    if (t_file == NULL) {
        return; 
    }
    
    transaction t;
    t.acc_no = acc_no;
    t.amount = amount;
    strcpy(t.type, type);
    
    // Get current system date and time
    time_t now = time(NULL);
    struct tm *t_struct = localtime(&now);
    strftime(t.date_time, sizeof(t.date_time), "%Y-%m-%d %H:%M:%S", t_struct);
    
    fwrite(&t, sizeof(transaction), 1, t_file);
    fclose(t_file);
}

void create_account() {
    account acc;
    FILE *file = fopen(ACCOUNT_FILE, "ab+");
    if (file == NULL) {
        printf("\nUnable to open file.\n");
        return;
    }

    while (getchar() != '\n'); // Clear input buffer

    printf("\nEnter your name: ");
    fgets(acc.name, sizeof(acc.name), stdin);
    int ind = strcspn(acc.name, "\n");
    acc.name[ind] = '\0';

    printf("Enter your account number: ");
    scanf("%d", &acc.acc_no);
    acc.balance = 0.0f;

    fwrite(&acc, sizeof(acc), 1, file);
    fclose(file);
    printf("\nAccount created successfully!\n");
}

void deposit_money() {
    FILE *file = fopen(ACCOUNT_FILE, "rb+");
    if (file == NULL) {
        printf("\nUnable to open file or no records found.\n");
        return;
    }
    int acc_no;
    float money;
    account acc_r;

    printf("\nEnter your account number: ");
    scanf("%d", &acc_no);
    printf("Enter amount to deposit: ");
    scanf("%f", &money);

    if (money <= 0) {
        printf("Error: Deposit amount must be greater than zero.\n");
        fclose(file);
        return;
    }

    while (fread(&acc_r, sizeof(acc_r), 1, file)) {
        if (acc_r.acc_no == acc_no) {
            acc_r.balance += money;
            fseek(file, -((long)sizeof(acc_r)), SEEK_CUR);
            fwrite(&acc_r, sizeof(acc_r), 1, file);
            fclose(file);
            
            log_transaction(acc_no, "Deposit", money);
            
            printf("Successfully deposited Rs.%.2f. New balance is Rs.%.2f\n", money, acc_r.balance);
            return;      
        }
    }
    fclose(file);
    printf("Money could not be deposited. Account no %d was not found.\n", acc_no);
}

void withdraw_money() {
    FILE *file = fopen(ACCOUNT_FILE, "rb+");
    if (file == NULL) {
        printf("\nUnable to open file or no records found.\n");
        return;
    }
    int acc_no;
    float money;
    account acc_r;

    printf("\nEnter your account number: ");
    scanf("%d", &acc_no);
    printf("Enter amount to withdraw: ");
    scanf("%f", &money);

    if (money <= 0) {
        printf("Error: Withdrawal amount must be greater than zero.\n");
        fclose(file);
        return;
    }

    while (fread(&acc_r, sizeof(acc_r), 1, file)) {
        if (acc_r.acc_no == acc_no) {
            if (acc_r.balance >= money) {
                acc_r.balance -= money;
                fseek(file, -((long)sizeof(acc_r)), SEEK_CUR);
                fwrite(&acc_r, sizeof(acc_r), 1, file);
                
                log_transaction(acc_no, "Withdrawal", money);
                
                printf("Successfully withdrawn Rs.%.2f. Remaining balance is Rs.%.2f\n", money, acc_r.balance);
            } else {
                printf("Insufficient balance!\n");
            }
            fclose(file);
            return;
        }
    }
    fclose(file);
    printf("Money could not be withdrawn. Account no %d was not found.\n", acc_no);
}

void transfer_money() {
    FILE *file = fopen(ACCOUNT_FILE, "rb+");
    if (file == NULL) {
        printf("\nUnable to open file or no records found.\n");
        return;
    }

    int sender_acc, receiver_acc;
    float amount;
    account sender, receiver;
    long sender_pos = -1, receiver_pos = -1;

    printf("\nEnter your (Sender) account number: ");
    scanf("%d", &sender_acc);
    printf("Enter destination (Receiver) account number: ");
    scanf("%d", &receiver_acc);
    printf("Enter amount to transfer: ");
    scanf("%f", &amount);

    if (amount <= 0) {
        printf("Error: Transfer amount must be greater than zero.\n");
        fclose(file);
        return;
    }
    if (sender_acc == receiver_acc) {
        printf("Error: Cannot transfer money to the same account.\n");
        fclose(file);
        return;
    }

    // Step 1: Find the positions of both accounts in the file
    while (fread(&sender, sizeof(account), 1, file)) {
        if (sender.acc_no == sender_acc) {
            sender_pos = ftell(file) - sizeof(account);
            break;
        }
    }

    rewind(file); // Go back to the beginning to search for receiver
    while (fread(&receiver, sizeof(account), 1, file)) {
        if (receiver.acc_no == receiver_acc) {
            receiver_pos = ftell(file) - sizeof(account);
            break;
        }
    }

    // Step 2: Validate accounts and balances before performing the transaction
    if (sender_pos == -1) {
        printf("Error: Sender account %d not found.\n", sender_acc);
    } else if (receiver_pos == -1) {
        printf("Error: Receiver account %d not found.\n", receiver_acc);
    } else if (sender.balance < amount) {
        printf("Transfer failed! Insufficient balance in sender account.\n");
    } else {
        // Step 3: Perform calculations
        sender.balance -= amount;
        receiver.balance += amount;

        // Write updated sender record
        fseek(file, sender_pos, SEEK_SET);
        fwrite(&sender, sizeof(account), 1, file);

        // Write updated receiver record
        fseek(file, receiver_pos, SEEK_SET);
        fwrite(&receiver, sizeof(account), 1, file);

        // Step 4: Log transaction histories for both accounts
        log_transaction(sender_acc, "Transfer Out", amount);
        log_transaction(receiver_acc, "Transfer In", amount);

        printf("\nSuccessfully transferred Rs.%.2f from Acc %d to Acc %d!\n", amount, sender_acc, receiver_acc);
    }

    fclose(file);
}

void check_balance() {
    FILE *file = fopen(ACCOUNT_FILE, "rb");
    if (file == NULL) {
        printf("\nUnable to open file or no records found.\n");
        return;
    }
    int acc_no;
    account acc_read;

    printf("\nEnter your account number: ");
    scanf("%d", &acc_no);

    while (fread(&acc_read, sizeof(acc_read), 1, file)) {
        if (acc_read.acc_no == acc_no) {
            printf("Your balance is Rs.%.2f\n", acc_read.balance);
            fclose(file);
            return;
        }
    }
    fclose(file);
    printf("\nAccount no %d was not found.\n", acc_no);
}

void delete_account() {
    FILE *file = fopen(ACCOUNT_FILE, "rb");
    if (file == NULL) {
        printf("\nUnable to open file or no records exist.\n");
        return;
    }

    FILE *temp_file = fopen("temp.dat", "wb");
    if (temp_file == NULL) {
        printf("\nCritical Error: Unable to complete deletion.\n");
        fclose(file);
        return;
    }

    int acc_no;
    account acc_r;
    int found = 0;
    char confirm;

    printf("\nEnter the account number to delete: ");
    scanf("%d", &acc_no);

    while (fread(&acc_r, sizeof(account), 1, file)) {
        if (acc_r.acc_no == acc_no) {
            found = 1;
            printf("\n--- Account Info ---");
            printf("\nName: %s", acc_r.name);
            printf("\nBalance: Rs.%.2f", acc_r.balance);
            
            printf("\n\nAre you sure you want to permanently delete this account? (y/n): ");
            while (getchar() != '\n'); 
            scanf("%c", &confirm);

            if (confirm == 'y' || confirm == 'Y') {
                printf("Processing deletion...\n");
            } else {
                fwrite(&acc_r, sizeof(account), 1, temp_file);
                found = 2; 
            }
        } else {
            fwrite(&acc_r, sizeof(account), 1, temp_file);
        }
    }

    fclose(file);
    fclose(temp_file);

    if (found == 0) {
        printf("\nAccount number %d was not found.\n", acc_no);
        remove("temp.dat");
    } else if (found == 2) {
        printf("\nDeletion canceled. Account remains safe.\n");
        remove("temp.dat");
    } else {
        remove(ACCOUNT_FILE);
        rename("temp.dat", ACCOUNT_FILE);
        printf("\nAccount deleted successfully!\n");
    }
}

void view_transaction_history() {
    FILE *file = fopen(TRANSACTION_FILE, "rb");
    if (file == NULL) {
        printf("\nNo transaction logs found yet.\n");
        return;
    }

    int acc_no;
    transaction t;
    int found = 0;

    printf("\nEnter account number to view history: ");
    scanf("%d", &acc_no);

    printf("\n==================== MINI STATEMENT (Acc: %d) ====================\n", acc_no);
    printf("%-22s | %-12s | %-12s\n", "Date & Time", "Type", "Amount");
    printf("-----------------------------------------------------------------\n");

    while (fread(&t, sizeof(transaction), 1, file)) {
        if (t.acc_no == acc_no) {
            printf("%-22s | %-12s | Rs.%-12.2f\n", t.date_time, t.type, t.amount);
            found = 1;
        }
    }
    
    if (!found) {
        printf("No transactions recorded for this account.\n");
    }
    printf("=================================================================\n");

    fclose(file);
}
