COC/CTO Management System v2 (Google Apps Script)
Overview
This Google Apps Script project provides a comprehensive system for managing Compensatory Overtime Credits (COC) and Compensatory Time Off (CTO) within an organization, utilizing Google Sheets as the database. It allows administrators to record overtime, generate certificates, manage employee balances using FIFO (First-In, First-Out), process CTO applications, and handle related administrative tasks. The system features a web-based UI built using HTML Service for user interaction directly within Google Sheets.
Key Features
Employee Management: Add, edit, and view employee details (including status). Supports historical COC data import during employee setup.
COC Recording:
Automated calculation of COC earned based on time entries and day type (Weekday, Weekend, Holiday - including special types like Half-day, No Work).
Supports recording multiple entries for a selected month.
Validation against monthly (40 hours) and total balance (120 hours) limits.
Monthly Certificate Generation:
Consolidates all pending COC records for a specific month into a single certificate.
Generates a Google Doc certificate based on a template.
Updates relevant records to 'Active' status and sets expiration dates (based on certificate issue date + validity period).
Updates the employee's ledger and detailed balance sheet.
CTO Application & Management:
Record CTO applications, validating against available COC balance.
Deducts hours from the employee's balance using FIFO logic, consuming the oldest earned COC first.
Allows administrators to view, filter, edit (pending/approved), and cancel (pending/approved) CTO applications.
Cancellation of approved CTO restores hours using reverse FIFO logic.
Employee Ledger:
Provides a detailed transaction history for each employee (COC earned, CTO used, expirations, adjustments, cancellations).
Displays current COC balance.
Shows a breakdown of the active balance based on FIFO (which earned hours are still available and when they expire).
Holiday Management: Add, edit, and delete holidays and special non-working days, which directly affect COC calculations. Supports various holiday types with specific rules (e.g., half-day).
Dashboard: Provides a quick overview of system statistics (employee counts, total balances, pending items, expiring COC). Includes quick action buttons for common tasks and a recent activity log.
Data Integrity & Expiration: Includes functions to check FIFO integrity and automatically mark expired COC entries.
Settings & Configuration: Manage system settings like COC validity period and certificate signatories.
System Components
The project is structured into several Apps Script (.gs) and HTML (.html) files:
Code.gs: Main entry point, global constants (database ID, column mappings), and onOpen() menu creation.
API_*.gs (Admin, COC, CTO, Employee, Ledger): Server-side functions callable from the HTML front-end. Handle data retrieval, updates, and business logic execution.
UIFunctions.gs: Functions to create the Google Sheets menu and display HTML dialogs (modals).
DataFunctions.gs: Core functions for interacting with the Google Sheets database (reading, writing, finding data). Includes employee CRUD operations.
BusinessLogic.gs: Contains the core rules and calculations, such as overtime calculation (calculateOvertimeForDate), FIFO consumption (consumeCOCWithFIFO), and application recording logic.
HelperFunctions.gs: Utility functions for date formatting, ID generation, retrieving settings, determining day types, etc.
Certificates.gs: Logic for generating the Google Doc certificates based on a template.
MigrationFunctions.gs: Scripts for one-time data migrations (e.g., populating MONTH_YEAR).
DebugAndTest.gs: Functions used for testing and diagnosing issues.
HTML Files (Dashboard.html, MonthlyCOCEntry.html, CTORecordForm.html, EmployeeLedger.html, EmployeeManager.html, HolidayManager.html, CTOApplicationsManager.html, etc.): Define the user interfaces presented in modal dialogs within Google Sheets. They use HTML, Tailwind CSS, and JavaScript (with google.script.run) to interact with the backend Apps Script functions.
Database Structure (Google Sheets)
The system relies on several sheets within a single Google Spreadsheet (DATABASE_ID defined in Code.gs):
Employees: Stores employee information (ID, Name, Position, Office, Status).
COC_Records: Logs individual overtime entries before they are certificated (Date Rendered, Times, Hours Worked, COC Earned, Status: Pending/Active/Cancelled).
COC_Certificates: Records generated monthly certificates (Certificate ID, Employee, Month/Year, Total Earned, Issue/Expiration Dates, Doc/PDF URLs).
COC_Balance_Detail: Detailed breakdown of all earned COC, used for FIFO tracking (Entry ID, Certificate/Record Link, Hours Earned/Used/Remaining, Expiration Date, Status: Active/Used/Expired/Depleted).
CTO_Applications: Logs all Compensatory Time Off requests (Application ID, Employee, Hours, Dates, Status: Pending/Approved/Cancelled).
COC_Ledger: Audit trail of all transactions affecting COC balance (Date, Type, Reference, Amount, Balance Before/After).
Holidays: List of official holidays and non-working days used for COC calculation.
Settings: Stores system configuration values (e.g., COC_VALIDITY_MONTHS, COC_CERTIFICATE_TEMPLATE_ID).
Signatories: Stores names and positions for certificate signatories.
Library: Contains lists for dropdowns (e.g., Positions, Offices).
Setup & Installation
Create a Google Sheet to act as the database.
Copy the Apps Script (.gs) and HTML (.html) files into the script editor bound to that Google Sheet.
Update the DATABASE_ID constant in Code.gs with the ID of your Google Sheet.
Configure a Google Doc template for COC certificates and update the COC_CERTIFICATE_TEMPLATE_ID in the Settings sheet. Ensure the script has permission to access this template.
Set up the necessary sheets (Employees, Holidays, Settings, Signatories, Library) with the correct headers as expected by the code (refer to column constants in Code.gs). The other sheets (COC_Records, COC_Certificates, COC_Balance_Detail, CTO_Applications, COC_Ledger) will be created or managed by the script.
Reload the Google Sheet to see the "COC/CTO System" menu.
Authorize the script when prompted.
Usage
Access the system functionalities through the "COC/CTO System" custom menu that appears in the Google Sheet interface upon opening.
Technical Details
Platform: Google Apps Script
Database: Google Sheets
Frontend: HTML Service, Tailwind CSS, JavaScript, jQuery, Select2
Core Logic: FIFO balance management, date-based calculations, automated certificate generation.
