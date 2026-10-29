# Data Dictionary

Source: `Banking_Analytics_Dataset.xlsx` (Accounts, Loans, Cards, SupportCalls) plus Customers and Transactions tables. The project works on a **sample** dataset.

## Tables

| Table | Rows | Grain | Columns |
|---|---|---|---|
| customers | 5,000 | One row per customer | CustomerID, full_name, JoinDate, State |
| Accounts | 5,000 | One row per account | AccountID, CustomerID, AccountType, Balance, CreatedDate |
| Transactions | 20,000 | One row per transaction | TransactionID, AccountID, TransactionDate, TransactionType, Amount, Season |
| Loans | 2,500 | One row per loan | LoanID, CustomerID, LoanType, LoanAmount, InterestRate, LoanStartDate, LoanEndDate |
| Cards | 4,000 | One row per card | CardID, CustomerID, CardType, IssuedDate, ExpirationDate |
| SupportCalls | 3,000 | One row per call | CallID, CustomerID, CallDate, IssueType, Resolved |
| DimDate | Calendar | One row per day | Date, Day, Day Name, Day Number, Month, Month Number, Month Year, Quarter, Season, Year, YearMonthNumber |

## Categorical values

| Column | Values |
|---|---|
| Accounts[AccountType] | Business, Checking, Savings |
| Transactions[TransactionType] | Deposit, Withdrawal, Transfer, Payment |
| Loans[LoanType] | Personal, Education, Car, Home |
| Cards[CardType] | Debit, Prepaid, Credit |
| SupportCalls[IssueType] | Transaction Dispute, Account Access, Card Issue, Loan Query |
| SupportCalls[Resolved] | Yes, No |

## Calculated columns

| Table | Column | Logic |
|---|---|---|
| DimDate | YearMonthNumber | Year x 100 + Month, used to sort `Month Year` chronologically |
| customers | Customer Last Txn | Latest transaction date across the customer's accounts |
| customers | Customer Status | Active (last transaction within 90 days of data end), Cooling (90 to 180 days), Churn Risk (more than 180 days, or an account with no transactions), No Account |

## Data model

Star-style model. `customers` is the central dimension (one-to-many to Accounts, Loans, Cards, SupportCalls). `Accounts` is one-to-many to `Transactions`.
`DimDate` has **one active relationship** (with `Transactions[TransactionDate]`). All other date relationships are inactive and used in measures with `USERELATIONSHIP` (customers[JoinDate], Accounts[CreatedDate], Cards[IssuedDate], Loans[LoanStartDate], Loans[LoanEndDate], SupportCalls[CallDate]).

## Data notes (verified against the source Excel)

- Last date in the data: **11 May 2025**. Active and churn rules are measured from this date, not from today.
- Support calls run from 11 May 2024 to 11 May 2025, so the first and last months in monthly charts are partial.
- All 2,500 loans are active at the data end date (loan end dates run from 12 May 2025 to 11 May 2030), so Active Loan Amount equals Total Loan Amount.
- All cards expire between 12 May 2026 and 11 May 2030, so no card is expired at the data end date.
- 3,171 customers hold at least one account; 1,829 customers have no account. Some customers hold more than one account (1,342).
- Totals: loans 616,658,893; account balances 249,370,795; transactions 100.11M; support calls 3,000 (49.3% resolved).
- No duplicate rows or null values were found in Accounts, Loans or SupportCalls, and no rows were removed in Power Query.
