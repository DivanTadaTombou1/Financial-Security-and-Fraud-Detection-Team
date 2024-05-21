# Financial-Security-and-Fraud-Detection-Team

## Problem:

The Financial Security and Risk Management Department faces the critical challenge of detecting fraudulent activities and ensuring the integrity of financial transactions within the organization. As the organisation handles a vast volume of financial transactions daily, manually identifying suspicious patterns or fraudulent behavior is arduous and time-consuming. Moreover, with the increasing sophistication of fraudulent techniques, traditional methods prove inadequate in safeguarding the organization's financial assets.

## Solution:

To address this challenge, I will develop a comprehensive Financial Transactions and Fraud Detection System using SQL analytics. By leveraging advanced SQL techniques such as Recursive Queries, Window Functions, Common Table Expressions (CTEs), and Indexing, I will create a robust system capable of:

1. **Detecting Suspicious Transaction Patterns:** Utilizing recursive queries and window functions to identify anomalous transaction behavior indicative of fraudulent activity.

2. **Analyzing Cash Flow Trends:** Employing window functions to analyze cash flow trends over time, enabling the identification of irregularities or potential risks.

3. **Computing Account Balances:** Implementing efficient algorithms to compute accurate account balances over time, facilitating real-time monitoring of financial health.

4. **Identifying High-Risk Accounts:** Utilizing transaction history and advanced analytics to identify high-risk accounts based on predefined criteria, enabling proactive risk mitigation measures.

By developing this sophisticated Financial Transactions and Fraud Detection System, I will significantly enhance the Financial Security and Risk Management Department's ability to detect and mitigate fraudulent activities, safeguarding the organization's financial interests and reputation.

```sql

WITH RecursiveTransactionHierarchy AS (
    SELECT 
        t1.transaction_id,
        t1.account_id,
        t1.transaction_amount,
        t1.transaction_date,
        1 AS level
    FROM Transactions t1
    JOIN (
        SELECT account_id, MAX(transaction_date) AS max_date
        FROM Transactions
        WHERE transaction_type = 'debit'
        GROUP BY account_id
    ) t2 ON t1.account_id = t2.account_id AND t1.transaction_date = t2.max_date

    UNION ALL

    SELECT 
        t1.transaction_id,
        t1.account_id,
        t1.transaction_amount,
        t1.transaction_date,
        rth.level + 1 AS level
    FROM Transactions t1
    JOIN RecursiveTransactionHierarchy rth ON t1.account_id = rth.account_id
    WHERE t1.transaction_type = 'debit'
    AND t1.transaction_date > DATE_SUB(rth.transaction_date, INTERVAL 1 DAY)
),
ComputedAccountBalances AS (
    SELECT
        rth.account_id,
        MAX(bal.current_balance) AS current_balance
    FROM (
        SELECT
            account_id,
            SUM(transaction_amount) OVER (PARTITION BY account_id ORDER BY transaction_date) AS current_balance
        FROM (
            SELECT 
                *,
                ROW_NUMBER() OVER (PARTITION BY account_id ORDER BY transaction_date) AS rn
            FROM Transactions
            WHERE transaction_type = 'debit'
        ) AS ranked_transactions
    ) AS bal
    JOIN RecursiveTransactionHierarchy rth ON bal.account_id = rth.account_id
    WHERE bal.rn = 1
    GROUP BY rth.account_id
),
HighRiskAccounts AS (
    SELECT
        ca.account_id,
        ca.current_balance,
        CASE WHEN ca.current_balance < 0 THEN 'High Risk' ELSE 'Normal' END AS risk_category,
        AVG(ca.current_balance) OVER (PARTITION BY a.branch_id) AS branch_avg_balance,
        ROW_NUMBER() OVER (ORDER BY ca.current_balance DESC) AS balance_rank
    FROM ComputedAccountBalances ca
    JOIN Accounts a ON ca.account_id = a.account_id
)
SELECT
    hra.account_id,
    hra.current_balance,
    hra.risk_category,
    hra.branch_avg_balance,
    hra.balance_rank,
    CONCAT('Alert: Account ', hra.account_id, ' has ', hra.risk_category, ' balance') AS alert_message,
    CURRENT_TIMESTAMP() AS alert_date
FROM HighRiskAccounts hra
WHERE hra.balance_rank <= 100
ORDER BY hra.current_balance DESC;
