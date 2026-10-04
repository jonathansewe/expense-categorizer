\# Expense Categorizer



A Python script that reads a CSV of transactions and prints total

spending per category, sorted highest to lowest, with a grand total.



\## How to run



Put `transactions.csv` in the same folder, then run:



`python categorize.py`



Requires Python 3. No external packages.



\## Sample output

```

$ python categorize.py

Rent: $3000.00

Food: $492.00

Shopping: $300.00

Entertainment: $25.00

Transportation: $14.25

Total: $3831.25

```



\## Notes



The source CSV has inconsistent spacing after commas, so some category

values arrive as " Food" instead of "Food". The script calls `.strip()`

on each one, which removes the surrounding whitespace so both spellings

count as the same category.



\## Possible extensions



\- SQLite storage and queries

\- A dashboard with charts

\- Automatic category assignment from the merchant name





