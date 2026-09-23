# Rakesh-Kothula-Phyllo-Assignment
Product Analyst Intern Assignment



Product Analyst Internship Assignment — Meridian API


**Task 1: Documentation Discrepancies**

Comparing API_DOCS.md with the response payloads reveals five key issues: 

1. **Invalid status Value (“refunded”)**

**Docs Say:** status is one of pending, shipped, delivered, cancelled.

**Data Does** (orders_page1.json → ord_1003): status is "**refunded**".

**Impact:** Breaks integrations using strict enum validation, causing runtime errors or skipped records.



2. **Inconsistent Monetary Units (Floats vs. Integer Cents)**

**Docs Say:** All monetary amounts are integers in the smallest currency unit (e.g., $54.70 is 5470).

**Data Does** (orders_page2.json → ord_1006): Amounts are floats in decimal dollars ("subtotal": 44.0, "total": 53.62).

**Impact:** Causes calculation **errors** or multiplies decimals by 100, turning $53.62 into $5,362.00. 



3. **email Field Nullability**

**Docs Say**: customer.email is "**Always present**".

**Data Does** (orders_page2.json → ord_1005): email is **null**.

**Impact:** Causes **NullPointerException or crashes** in systems relying on email for customer identification, receipts, or notification services.



4. **Premature has_more: false Flag on Paginated Response**

**Docs Say:** Check has_more to decide whether to fetch the next page using next_cursor or not.

**Data Does** (orders_page1.json): has_more is false, but a valid next_cursor ("cur_8f2a19bd") is returned alongside a second page of data (orders_page2.json).

**Impact:** Automated scripts stop paginating early, **silently missing page 2 records** (ord_1005 and ord_1006).



5. **404 Not Found Missing Object Schema**

**Docs Say:** GET /v1/orders/{id} returns 404 if no order exists.

**Data Does** (order_ord_9999.json): Request returned HTTP 200 with {"order": null}.

**Impact:** When an order doesn't exist, the API sends a "Success" code (HTTP 200 OK) instead of "Not Found" (404). This tricks code into thinking an order was found, so it tries to read details inside null and crashes. 



**Worst Finding:** Premature has_more: **false**

This pagination bug is the most critical because it causes silent data loss. Unlike crashes, scripts blindly trust has_more: false and **drop remaining records without raising any alert**. 



—-----------------------------------------------------------------------------------------------------------



**Task 2: Total Revenue Calculation**

Total Revenue: $274.08 (Gross) | $171.75 (Net) 

Analysis & Breakdown


| Order ID | Status    | Amount in Data | Converted USD | Reasoning               |
| :---     | :---      | :---           | :---          | :---                    |
| ord_1001 | shipped   | 5470           | $54.70        | Included (Cents to USD) |
| ord_1002 | delivered | 2381           | $23.81        | Included (Cents to USD) |
| ord_1003 | refunded  | 10233          | $102.33       | Excluded from Net |
| ord_1004 | shipped   | 6810           | $68.10        | Included (Cents to USD) |
| ord_1005 | delivered | 2547           | $25.47        | Included (Cents to USD) |
| ord_1006 | shipped   | 53.62          | $53.62        | Included (Already decimal dollars) |



**Key Assumptions & Decisions:**

1 **.Decimal Handling (ord_1006):** Counted as $53.62 directly rather than 5362 cents to keep calculations accurate. 

2. **Refunded Orders (ord_1003):** Treated as returned revenue ($102.33 deducted for Net Revenue) 
3. **Data Needed for Certainty:** Confirmation from engineering on currency formatting, and guidance from accounting on reporting Gross vs Net totals. 




—------------------------------------------------------------------------------------------------------------




**Task 3: Written Deliverables**



**Email Reply to Priya**

Subject: Re: Revenue reconciliation discrepancy

Hi Priya,

Thanks for bringing this up! The discrepancy between your monthly report and the Meridian dashboard is caused by two main API issues: 

1. **Pagination Cutoff:** The API response for page 1 returns **"has_more": false** even though additional **orders exist on page 2**. If your script stops fetching when has_more is false, it drops all subsequent orders from page 2 onward.


2. **Data Format Inconsistencies:** Most orders express totals in cents (e.g., 5470 = $54.70), but some recent records return raw decimal amounts (e.g., 53.62 = $53.62). **Summing these directly inflates totals or causes script errors**. Additionally, order ord_1003 ($102.33) is marked as **refunded** and should be **excluded from net revenue**.


**Next Steps:**
Please update your **integration to fetch page 2** using cursor cur_8f2a19bd and handle **float values**. We are working with Engineering to fix these API issues.

Thanks & Regards,
Rakesh Kothula
Product Analyst Team 






**Bug Report**

**Title:** GET /v1/orders returns has_more: false despite active next_cursor and **page 2 data**

**Severity:** **High** (Causes silent data issues in downstream systems)

**Environment:** Production (GET /v1/orders) 

Description:

When querying page 1 of the orders list endpoint, the response payload sets "has_more": false despite containing a **valid "next_cursor"** ("cur_8f2a19bd") and additional records on page 2.

Steps to Reproduce:

1. Send GET /v1/orders.

2. Inspect the returned JSON payload (orders_page1.json) in the network tab.

3. Observe "has_more": false and "next_cursor": "cur_8f2a19bd".

4. Send GET /v1/orders?starting_after=cur_8f2a19bd.


**Expected Behavior:**

has_more should evaluate to true whenever additional pages exist.

**Actual Behavior:**

has_more returns false, causing client pagination loops to terminate early. 

**Proposed Fix:**

Fix the API pagination code to check if remaining items exist before setting has_more to false.
