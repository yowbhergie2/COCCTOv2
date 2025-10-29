// -----------------------------------------------------------------------------
// BusinessLogic.gs
// -----------------------------------------------------------------------------

/**
 * Records one or more overtime entries for a single employee and month. Entries
 * are validated against monthly limits, duplicate dates and maximum overall
 * balance constraints. If any entry fails validation the entire operation
 * aborts and no rows are appended.
 *
 * @param {string} employeeId The employee ID.
 * @param {number} month The month number (1–12).
 * @param {number} year The year (four digits).
 * @param {Array<Object>} entries List of objects {day, amIn, amOut, pmIn, pmOut}.
 * @return {Object} Summary of the operation including how many rows were added,
 * total COC earned and the resulting balance.
 */
function recordCOCEntries(employeeId, month, year, entries) {
  const db = getDatabase();
  const recordsSheet = db.getSheetByName('COC_Records');
  const ledgerSheet = db.getSheetByName('COC_Ledger');
  if (!recordsSheet || !ledgerSheet) {
    throw new Error('Required sheets missing');
  }
  const settings = getSettings();
  const employee = getEmployeeById(employeeId);
  if (!employee) {
    throw new Error('Employee not found');
  }
  const TIME_ZONE = getScriptTimeZone(); // Added this
  // Calculate existing hours for this month
  let monthlyTotal = 0;
  // Use MM-yyyy format consistently
  const monthYear = String(month).padStart(2, '0') + '-' + String(year);
  const recData = recordsSheet.getDataRange().getValues();
  for (let i = 1; i < recData.length; i++) {
    const row = recData[i];
    if (row[1] === employeeId && String(row[3]).trim() === monthYear) {
      monthlyTotal += parseFloat(row[10]) || 0;
    }
  }
  // Prepare new rows and ledger entries but do not write yet
  let totalNewHours = 0;
  let totalNewCOC = 0;
  let runningBalance = getCurrentCOCBalance(employeeId);
  const newRows = [];
  const newLedgerRows = [];
  // Duplicate detection: collect existing date strings for this employee.
  // Skip rows that have been cancelled so that users may re‑enter a date
  // after cancelling a previous record. Status is in column 16 (index 15).

  const existingDates = {};
  const existingRecordDetails = {};

  for (let i = 1; i < recData.length; i++) {
    const row = recData[i];
    if (row[1] === employeeId) {
      const status = row[15];
      if (status && String(status).toLowerCase() === 'cancelled') continue;
      const d = new Date(row[4]);
      const key = Utilities.formatDate(d, TIME_ZONE, 'yyyy-MM-dd');
      existingDates[key] = true;
      existingRecordDetails[key] = {
        recordId: row[0],
        date: formatDate(d),
        dayType: row[5],
        hoursWorked: row[10],
        cocEarned: row[11]
      };
    }
  }

  // Check for duplicates and collect them
  const duplicates = [];
  entries.forEach(entry => {
    const date = new Date(year, month - 1, entry.day);
    const key = Utilities.formatDate(date, TIME_ZONE, 'yyyy-MM-dd');
    if (existingDates[key]) {
      duplicates.push({
        date: formatDate(date),
        existing: existingRecordDetails[key]
      });
    }
    const result = calculateOvertimeForDate(date, entry.amIn, entry.amOut, entry.pmIn, entry.pmOut);
    totalNewHours += result.hoursWorked;
    totalNewCOC += result.cocEarned;
  });

  // If duplicates found, throw detailed error
  if (duplicates.length > 0) {
    let errorMsg = 'The following date(s) already have COC records:\n\n';
    duplicates.forEach(dup => {
      errorMsg += '• ' + dup.date + ' (' + dup.existing.dayType + ', ' +
                    dup.existing.hoursWorked + ' hours, ' +
                    dup.existing.cocEarned + ' COC)\n';
    });
    errorMsg += '\nTo update these records:\n';
    errorMsg += '1. Remove these entries from your current form, OR\n';
    errorMsg += '2. Delete the existing records from the database first, then resubmit\n\n';
    errorMsg += 'Note: You cannot have multiple entries for the same date.';
    throw new Error(errorMsg);
  }
  // Enforce monthly hours limit
  const maxPerMonth = parseFloat(settings['MAX_COC_PER_MONTH']) || 40;
  if (monthlyTotal + totalNewHours > maxPerMonth) {
    throw new Error('Monthly COC limit of ' + maxPerMonth + ' hours exceeded');
  }
  // Enforce total balance limit
  const maxBalance = parseFloat(settings['MAX_COC_BALANCE']) || 120;
  if (runningBalance + totalNewCOC > maxBalance) {
    throw new Error('Total COC balance cannot exceed ' + maxBalance + ' hours');
  }
  // Now build rows
  entries.forEach(entry => {
    const date = new Date(year, month - 1, entry.day);
    const result = calculateOvertimeForDate(date, entry.amIn, entry.amOut, entry.pmIn, entry.pmOut);
    if(result.cocEarned <= 0) return; // Don't log entries with 0 hours

    runningBalance += result.cocEarned;
    const recordId = generateRecordId();
    newRows.push([
      recordId,
      employeeId,
      employee.fullName,
      monthYear,
      date,
      result.dayType,
      entry.amIn || '',
      entry.amOut || '',
      entry.pmIn || '',
      entry.pmOut || '',
      result.hoursWorked,
      result.multiplier,
      result.cocEarned,
      new Date(),
      '',
      'Active'
    ]);
    // Mirror the earned entry in the FIFO detail sheet so that CTO consumption
    // and expiration tracking have a single source of truth. Certificate details
    // are filled in once the consolidated certificate is issued.
    addCOCToBalanceDetail(
      employeeId,
      employee.fullName,
      recordId,
      date,
      result.cocEarned
    );
    const ledgerId = generateLedgerId();
    newLedgerRows.push([
      ledgerId,
      employeeId,
      employee.fullName,
      new Date(),
      'COC Earned',
      recordId,
      result.cocEarned,
      0,
      runningBalance,
      monthYear,
      '',
      Session.getActiveUser().getEmail(),
      'COC entry for ' + formatDate(date)
    ]);
  });
  // Append rows to both sheets
  if (newRows.length > 0) {
    recordsSheet.getRange(recordsSheet.getLastRow() + 1, 1, newRows.length, newRows[0].length).setValues(newRows);
    ledgerSheet.getRange(ledgerSheet.getLastRow() + 1, 1, newLedgerRows.length, newLedgerRows[0].length).setValues(newLedgerRows);
  }
  return {
    added: newRows.length,
    totalNewCOC: totalNewCOC,
    balanceAfter: runningBalance
  };
}


/**
 * Records a CTO application for an employee. Validation ensures hours are a
 * multiple of four, requested days do not exceed five consecutive days,
 * sufficient balance exists, and then writes both the application and ledger
 * entries.
 *
 * @param {string} employeeId The employee ID.
 * @param {number} hours The number of CTO hours requested (must be multiple of 4).
 * @param {Date} startDate The first day of CTO.
 * @param {Date} endDate The last day of CTO.
 * @param {string} remarks Optional remarks.
 * @return {Object} Contains the generated application ID and the new balance.
*/
function recordCTOApplication(employeeId, hours, startDate, endDate, remarks) {
  const db = getDatabase();
  const ctoSheet = db.getSheetByName('CTO_Applications');
  const ledgerSheet = db.getSheetByName('COC_Ledger');
  if (!ctoSheet || !ledgerSheet) throw new Error('Required sheets missing');
  const employee = getEmployeeById(employeeId);
  if (!employee) throw new Error('Employee not found');
  hours = parseFloat(hours);
  if (hours <= 0 || hours % 4 !== 0) throw new Error('CTO hours must be a positive multiple of 4');
  // If the request is for a half‑day (4hrs) or a full day (8hrs) the start and end
  // dates must be the same. This prevents the user from spanning multiple days
  // with only a partial day of CTO, which is prohibited by the business rules.
  if ((hours === 4 || hours === 8) && startDate.getTime() !== endDate.getTime()) {
    throw new Error('For 4 or 8 hour CTO applications the start and end date must be the same');
  }
  // Check consecutive days (inclusive)
  const daysRequested = Math.floor((endDate.getTime() - startDate.getTime()) / (1000 * 60 * 60 * 24)) + 1;
  if (daysRequested > 5) throw new Error('Maximum consecutive CTO days is 5');
  const currentBalance = getCurrentCOCBalance(employeeId);
  if (currentBalance < hours) throw new Error('Insufficient COC balance');
  const inclusiveDates = formatDate(startDate) + ' - ' + formatDate(endDate);
  const ctoId = generateCTOId();
  // Record CTO application
  ctoSheet.appendRow([
    ctoId,
    employeeId,
    employee.fullName,
    employee.office,
    hours,
    startDate,
    endDate,
    inclusiveDates,
    currentBalance,
    new Date(),
    'Approved',
    new Date(),
    remarks || ''
  ]);
  // Update ledger
  const ledgerId = generateLedgerId();
  const newBalance = currentBalance - hours;
  ledgerSheet.appendRow([
    ledgerId,
    employeeId,
    employee.fullName,
    new Date(),
    'CTO Used',
    ctoId,
    0,
    hours,
    newBalance < 0 ? 0 : newBalance,
    '',
    '',
    Session.getActiveUser().getEmail(),
    remarks || 'CTO application'
  ]);
  return {
    applicationId: ctoId,
    balanceAfter: newBalance
  };
}


/**
 * MODIFIED: Enhanced recordCTOApplication with FIFO
 * This should replace or enhance the existing recordCTOApplication function
*/
function recordCTOApplicationWithFIFO(employeeId, hours, startDate, endDate, remarks) {
  // Ensure startDate and endDate are Date objects
startDate = new Date(startDate);
endDate   = new Date(endDate);
const db = getDatabase();
  const employee = getEmployeeById(employeeId);
  if (!employee) throw new Error('Employee not found');

  const balance = getCurrentCOCBalance(employeeId);
  if (hours > balance) {
    throw new Error('Insufficient COC balance. Available: ' + balance.toFixed(2) +
                      ' hours, Requested: ' + hours.toFixed(2) + ' hours');
  }

  const ctoId = generateCTOId();
  const ledgerId = generateLedgerId();
  const inclusiveDates = formatDate(startDate) + ' to ' + formatDate(endDate);
  const TIME_ZONE = getScriptTimeZone();

  // Apply FIFO consumption
  const deductions = consumeCOCWithFIFO(employeeId, hours, ctoId);

  // Create detailed remarks showing FIFO consumption
  const fifoDetails = deductions.map(d =>
    '  • ' + d.hoursDeducted.toFixed(2) + ' hrs from ' + d.recordId +
    ' (earned ' + d.dateEarned + ')'
  ).join('\n');

  const detailedRemarks = (remarks || 'CTO application') + '\n\nFIFO Consumption:\n' + fifoDetails;

  // Record in CTO_Applications
  const ctoSheet = db.getSheetByName('CTO_Applications');
  if (ctoSheet) {
    ctoSheet.appendRow([
      ctoId,
      employeeId,
      employee.fullName,
      employee.office,
      hours,
      startDate,
      endDate,
      inclusiveDates,
      balance,
      new Date(),
      'Approved',
      new Date(),
      detailedRemarks
    ]);
  }

  // Record in COC_Ledger (one entry per FIFO deduction for audit trail)
  const ledgerSheet = db.getSheetByName('COC_Ledger');
  if (ledgerSheet) {
    deductions.forEach(deduction => {
      const deductionLedgerId = generateLedgerId();
      const monthYear = deduction.dateEarned.substring(0, 7).split('-').reverse().join('-'); // Convert to MM-yyyy

      ledgerSheet.appendRow([
        deductionLedgerId,
        employeeId,
        employee.fullName,
        new Date(),
        'CTO Used',
        ctoId,
        0, // COC Earned
        deduction.hoursDeducted, // CTO Used
        balance - hours, // New balance
        monthYear,
        '', // Expiration (not applicable for usage)
        Session.getActiveUser().getEmail(),
        'FIFO: Used ' + deduction.hoursDeducted.toFixed(2) + ' hrs from ' +
        deduction.recordId + ' (earned ' + deduction.dateEarned + ') for ' + ctoId
      ]);
    });
  }

  return {
    success: true,
    ctoId: ctoId,
    hoursUsed: hours,
    newBalance: balance - hours,
    fifoDeductions: deductions
  };
}


/**
 * Cancels a CTO application and credits the hours back to the employee’s
 * balance. Only applications with status "Approved" can be cancelled.
 * A new ledger entry is created to record the reversal. The CTO record
 * status is set to "Cancelled" and the approval date is preserved.
 *
 * @param {string} ctoId The application ID (e.g. CTO‑20250101123000).
 * @param {string} remarks Optional remarks explaining the cancellation.
 * @return {Object} Contains the new balance after cancellation.
 */
function cancelCTOApplication(ctoId, remarks) {
  const db = getDatabase();
  const ctoSheet = db.getSheetByName('CTO_Applications');
  const ledgerSheet = db.getSheetByName('COC_Ledger');
  if (!ctoSheet || !ledgerSheet) throw new Error('Required sheets missing');
  const data = ctoSheet.getDataRange().getValues();
  let rowIndex = -1;
  let record = null;
  for (let i = 1; i < data.length; i++) {
    if (data[i][0] === ctoId) {
      rowIndex = i + 1;
      record = data[i];
      break;
    }
  }
  if (rowIndex === -1) throw new Error('CTO application not found');
  // record fields: [0]=ID, [1]=employeeId, [2]=employeeName, [3]=office,
  // [4]=hours, [5]=startDate, [6]=endDate, [7]=inclusiveDates,
  // [8]=balanceBefore, [9]=applicationDate, [10]=status, [11]=approvalDate, [12]=remarks
  const status = record[10];
  if (status !== 'Approved') throw new Error('Only approved CTO applications can be cancelled');
  const employeeId = record[1];
  const hours = parseFloat(record[4]) || 0;
  // Update CTO status
  ctoSheet.getRange(rowIndex, 11).setValue('Cancelled');
  ctoSheet.getRange(rowIndex, 13).setValue(remarks || 'CTO cancelled');
  // Credit back hours: new balance = current + hours
  const currentBalance = getCurrentCOCBalance(employeeId);
  const newBalance = currentBalance + hours;
  // Create ledger reversal entry
  const ledgerId = generateLedgerId();
  const employee = getEmployeeById(employeeId);
  ledgerSheet.appendRow([
    ledgerId,
    employeeId,
    employee.fullName,
    new Date(),
    'CTO Cancelled',
    ctoId,
    0,
    -hours,
    newBalance,
    '',
    '',
    Session.getActiveUser().getEmail(),
    remarks || 'CTO cancellation'
  ]);
  return { balanceAfter: newBalance };
}


/**
 * Cancels a COC record. The record status is set to "Cancelled" and a
 * reversal ledger entry is created. Users may then re‑record a new COC
 * entry for the same date. The original COC earned hours are deducted
 * from the employee’s balance.
 *
 * @param {string} recordId The COC record ID.
 * @param {string} remarks Optional remarks explaining the cancellation.
 * @return {Object} Contains the new balance after cancellation.
 */
function cancelCOCRecord(recordId, remarks) {
  const db = getDatabase();
  const recSheet = db.getSheetByName('COC_Records');
  const ledgerSheet = db.getSheetByName('COC_Ledger');
  if (!recSheet || !ledgerSheet) throw new Error('Required sheets missing');
  const data = recSheet.getDataRange().getValues();
  let rowIndex = -1;
  let record = null;
  for (let i = 1; i < data.length; i++) {
    if (data[i][0] === recordId) {
      rowIndex = i + 1;
      record = data[i];
      break;
    }
  }
  if (rowIndex === -1) throw new Error('COC record not found');
  const status = record[15];
  // Only active records can be cancelled
  if (status === 'Cancelled') throw new Error('COC record is already cancelled');
  // Mark record as cancelled
  recSheet.getRange(rowIndex, 16).setValue('Cancelled');
  // Deduct COC earned
  const employeeId = record[1];
  const cocEarned = parseFloat(record[12]) || 0;
  const currentBalance = getCurrentCOCBalance(employeeId);
  const newBalance = currentBalance - cocEarned;
  const employee = getEmployeeById(employeeId);
  // Append reversal in ledger
  const ledgerId = generateLedgerId();
  ledgerSheet.appendRow([
    ledgerId,
    employeeId,
    employee.fullName,
    new Date(),
    'COC Cancelled',
    recordId,
    -cocEarned,
    0,
    newBalance,
    record[3], // Month‑Year Earned
    record[14], // Expiration Date
    Session.getActiveUser().getEmail(),
    remarks || 'COC cancellation'
  ]);
  return { balanceAfter: newBalance };
}


/**
 * FIFO COC Consumption
 * Returns array of deductions showing which entries were consumed
 */
function consumeCOCWithFIFO(employeeId, hoursToConsume, reference) {
  const detailSheet = ensureCOCBalanceDetailSheet();
  const data = detailSheet.getDataRange().getValues();
  const TIME_ZONE = getScriptTimeZone();

  // Get all active entries for this employee
  const availableEntries = [];

  for (let i = 1; i < data.length; i++) {
    const row = data[i];
    const status = String(row[DETAIL_COLS.STATUS] || '').trim();
    const remaining = parseFloat(row[DETAIL_COLS.HOURS_REMAINING]) || 0;
    if (row[DETAIL_COLS.EMPLOYEE_ID] === employeeId && status === 'Active' && remaining > 0) {
      const certificateDateVal = row[DETAIL_COLS.CERTIFICATE_DATE];
      const orderingDate = (certificateDateVal instanceof Date && !isNaN(certificateDateVal.getTime()))
        ? new Date(certificateDateVal)
        : (parseMonthYear(row[DETAIL_COLS.MONTH_YEAR]) || new Date(row[DETAIL_COLS.DATE_CREATED]));

      const expirationVal = row[DETAIL_COLS.EXPIRATION_DATE];
      const expirationDate = (expirationVal instanceof Date && !isNaN(expirationVal.getTime()))
        ? new Date(expirationVal)
        : null;

      availableEntries.push({
        rowIndex: i + 1,
        recordId: row[DETAIL_COLS.RECORD_ID],
        dateReference: orderingDate,
        hoursEarned: parseFloat(row[DETAIL_COLS.HOURS_EARNED]) || 0,
        hoursUsed: parseFloat(row[DETAIL_COLS.HOURS_USED]) || 0,
        hoursRemaining: remaining,
        expirationDate: expirationDate
      });
    }
  }

  // Sort by certificate date (FIFO - oldest first)
  availableEntries.sort((a, b) => a.dateReference - b.dateReference);

  // Calculate total available
  const totalAvailable = availableEntries.reduce((sum, entry) => sum + entry.hoursRemaining, 0);

  if (hoursToConsume > totalAvailable) {
    throw new Error('Insufficient COC balance. Available: ' + totalAvailable.toFixed(2) +
                      ' hours, Requested: ' + hoursToConsume.toFixed(2) + ' hours');
  }

  // Apply FIFO deduction
  let remainingToConsume = hoursToConsume;
  const deductions = [];

  for (const entry of availableEntries) {
    if (remainingToConsume <= 0) break;

    const deductFromThis = Math.min(remainingToConsume, entry.hoursRemaining);
    const newRemaining = entry.hoursRemaining - deductFromThis;
    const newUsed = entry.hoursUsed + deductFromThis;
    const timestamp = new Date();
    const note = `[${Utilities.formatDate(timestamp, TIME_ZONE, 'yyyy-MM-dd HH:mm')}] Consumed ${deductFromThis.toFixed(2)} hrs for ${reference}`;

    detailSheet.getRange(entry.rowIndex, DETAIL_COLS.HOURS_USED + 1).setValue(newUsed);
    detailSheet.getRange(entry.rowIndex, DETAIL_COLS.HOURS_REMAINING + 1).setValue(newRemaining);
    if (newRemaining === 0) {
      detailSheet.getRange(entry.rowIndex, DETAIL_COLS.STATUS + 1).setValue('Depleted');
    }

    const existingNote = detailSheet.getRange(entry.rowIndex, DETAIL_COLS.REMARKS + 1).getValue() || '';
    const combinedNote = existingNote ? existingNote + '\n' + note : note;
    detailSheet.getRange(entry.rowIndex, DETAIL_COLS.REMARKS + 1).setValue(combinedNote);

    deductions.push({
      recordId: entry.recordId,
      hoursDeducted: deductFromThis,
      remainingHours: newRemaining
    });

    remainingToConsume -= deductFromThis;
  }

  return deductions;
}


function deductCOCHoursFIFO(employeeId, hoursToDeduct, referenceId) {
  const db = getDatabase();
  const detailSheet = db.getSheetByName('COC_Balance_Detail');
  // --- MODIFICATION: Added Ledger Sheet ---
  const ledgerSheet = db.getSheetByName('COC_Ledger');
  // --- END MODIFICATION ---

  if (!detailSheet) throw new Error('COC_Balance_Detail sheet not found');
  // --- MODIFICATION: Added Ledger Sheet Check ---
  if (!ledgerSheet) throw new Error('COC_Ledger sheet not found');
  // --- END MODIFICATION ---


  const data = detailSheet.getDataRange().getValues();
  let remainingToDeduct = hoursToDeduct;
  const TIME_ZONE = getScriptTimeZone(); // Get timezone
  const updatesToDetailSheet = []; // For batch updates
  const ledgerEntries = []; // For batch ledger updates

  Logger.log(`FIFO Deduction: Attempting to deduct ${hoursToDeduct} hrs for ${employeeId}, reference: ${referenceId}`);


  // Get active entries for this employee, sorted by date (FIFO)
  const activeEntries = [];
  for (let i = 1; i < data.length; i++) {
    const row = data[i];
    // Check Employee ID, Status 'Active', and Hours Remaining > 0
    if (row[1] === employeeId && row[8] === 'Active' && (parseFloat(row[6]) || 0) > 0) {
      activeEntries.push({
        rowIndex: i + 1, // 1-based index
        entryId: row[0],
        recordId: row[3],
        dateEarned: new Date(row[4]),
        hoursRemaining: parseFloat(row[6]) || 0
      });
    }
  }

  // Sort by date earned (oldest first - FIFO)
  activeEntries.sort((a, b) => a.dateEarned - b.dateEarned);

   // Check if enough balance exists before starting deductions
    const totalAvailable = activeEntries.reduce((sum, entry) => sum + entry.hoursRemaining, 0);
    if (hoursToDeduct > totalAvailable) {
        Logger.log(`FIFO Deduction Error: Insufficient balance. Available: ${totalAvailable.toFixed(2)}, Needed: ${hoursToDeduct.toFixed(2)}`);
        throw new Error(`Insufficient COC balance for deduction. Available: ${totalAvailable.toFixed(2)}, Requested: ${hoursToDeduct.toFixed(2)}`);
    }


  // Deduct from entries
  activeEntries.forEach(entry => {
    if (remainingToDeduct <= 0) return;

    const hoursFromThisEntry = Math.min(entry.hoursRemaining, remainingToDeduct);
    const newRemaining = entry.hoursRemaining - hoursFromThisEntry;

    // Prepare detail sheet updates
    updatesToDetailSheet.push({ row: entry.rowIndex, col: 7, value: newRemaining }); // Hours Remaining (Col G)
     Logger.log(`FIFO Deduction: Using ${hoursFromThisEntry.toFixed(2)} hrs from ${entry.recordId} (Row ${entry.rowIndex}). New remaining: ${newRemaining.toFixed(2)}`);


    if (newRemaining <= 0) {
      updatesToDetailSheet.push({ row: entry.rowIndex, col: 9, value: 'Used' }); // Status (Col I) - Mark as 'Used' instead of 'Depleted' for clarity
       Logger.log(`   Marking ${entry.recordId} (Row ${entry.rowIndex}) as Used.`);

    }
     // Add note about consumption (Column K = index 11)
    const currentNote = detailSheet.getRange(entry.rowIndex, 11).getValue(); // Get current note value
    const newNote = currentNote + '\n[' + Utilities.formatDate(new Date(), TIME_ZONE, 'yyyy-MM-dd HH:mm') +
                      '] Consumed ' + hoursFromThisEntry.toFixed(2) + ' hrs for ' + referenceId;
    updatesToDetailSheet.push({ row: entry.rowIndex, col: 11, value: newNote }); // Notes (Col K)



    // --- MODIFICATION: Prepare ledger entry INSTEAD of writing directly ---
    // We add this later AFTER calculating the final balance
    const ledgerRemark = `FIFO: Used ${hoursFromThisEntry.toFixed(2)} hrs from ${entry.recordId} (earned ${Utilities.formatDate(entry.dateEarned, TIME_ZONE, 'yyyy-MM-dd')}) for ${referenceId}`;
    ledgerEntries.push({
        employeeId: employeeId,
        // employeeName will be filled later if needed, or retrieved via getEmployeeById
        transactionDate: new Date(),
        transactionType: 'CTO Used (FIFO)', // More specific type
        referenceId: referenceId, // Link to the CTO or update action
        cocEarned: 0,
        ctoUsed: hoursFromThisEntry, // Log the specific amount deducted
        // cocBalance will be calculated and added later
        monthYearEarned: '', // Not applicable for usage
        expirationDate: '', // Not applicable for usage
        processedBy: Session.getActiveUser().getEmail(),
        remarks: ledgerRemark
    });
    // --- END MODIFICATION ---

    remainingToDeduct -= hoursFromThisEntry;
  });

   // Apply batch updates to the detail sheet
    updatesToDetailSheet.forEach(update => {
        detailSheet.getRange(update.row, update.col).setValue(update.value);
    });
    Logger.log(`FIFO Deduction: Applied ${updatesToDetailSheet.length} updates to COC_Balance_Detail.`);


  // --- MODIFICATION: Add Ledger Entries with correct running balance ---
   if (ledgerEntries.length > 0) {
        const employeeData = getEmployeeById(employeeId); // Get employee name
        const employeeName = employeeData ? employeeData.fullName : 'Unknown Employee';
        const finalBalance = getCurrentCOCBalance(employeeId); // Get final balance AFTER deductions are applied to detail sheet

        const rowsToAdd = ledgerEntries.map((entry, index) => {
             // Calculate the running balance for each entry (might be slightly off if multiple entries, but good for audit)
             // A more accurate way is to just put the final balance on all entries for this transaction.
            return [
                generateLedgerEntryId(),
                entry.employeeId,
                employeeName, // Add employee name
                entry.transactionDate,
                entry.transactionType,
                entry.referenceId,
                entry.cocEarned,
                entry.ctoUsed,
                finalBalance, // Use the final balance for all related entries
                entry.monthYearEarned,
                entry.expirationDate,
                entry.processedBy,
                entry.remarks
            ];
        });

        // Batch write ledger entries
        ledgerSheet.getRange(ledgerSheet.getLastRow() + 1, 1, rowsToAdd.length, rowsToAdd[0].length).setValues(rowsToAdd);
        Logger.log(`FIFO Deduction: Added ${rowsToAdd.length} ledger entries.`);
    }
  // --- END MODIFICATION ---

  // This check should ideally not be needed if balance check is done before calling
  if (remainingToDeduct > 0.01) { // Allow tiny floating point differences
    Logger.log(`FIFO Deduction Error: Could not deduct all required hours. ${remainingToDeduct.toFixed(2)} hrs remaining.`);
    throw new Error('Insufficient COC balance detected during FIFO deduction.');
  }
   Logger.log(`FIFO Deduction: Successfully deducted ${hoursToDeduct.toFixed(2)} hrs.`);
}


/**
 * Restore COC hours to employee balance
 * Used when decreasing CTO hours
 */
function restoreCOCHoursFIFO(employeeId, hoursToRestore, referenceId) {
  const db = getDatabase();
  const detailSheet = db.getSheetByName('COC_Balance_Detail');
  const ledgerSheet = db.getSheetByName('COC_Ledger'); // Needed to find original deductions
  
  if (!detailSheet) throw new Error('COC_Balance_Detail sheet not found');
  if (!ledgerSheet) throw new Error('COC_Ledger sheet not found'); // Added check

  Logger.log(`Attempting to restore ${hoursToRestore} hours for ${employeeId} related to ${referenceId}`);
  
  // Find the ledger entries representing the *original* FIFO deductions for this referenceId
  const ledgerData = ledgerSheet.getDataRange().getValues();
  const originalDeductions = [];
  
  // Search from newest to oldest for CTO Used or FIFO Adjustment entries
  for (let i = ledgerData.length - 1; i >= 1; i--) {
    const row = ledgerData[i];
    // Check Employee ID, Reference ID, and Transaction Type or Remarks
    if (row[1] === employeeId && row[5] === referenceId) {
       // Look for "CTO Used" or "FIFO Adjustment" or remarks containing "FIFO: Used"
       const type = row[4];
       const remarks = row[12] || '';
       if (type === 'CTO Used' || type === 'FIFO Adjustment' || remarks.startsWith('FIFO: Used')) {
          // Extract hours and source record ID from remarks if possible
          const remarksMatch = remarks.match(/Used\s+([\d.]+)\s+hrs\s+from\s+(\S+)/);
          if (remarksMatch) {
            originalDeductions.push({
              hours: parseFloat(remarksMatch[1]),
              recordId: remarksMatch[2] // This is the Record ID from COC_Balance_Detail
            });
             Logger.log(`Found original deduction: ${remarksMatch[1]} hrs from ${remarksMatch[2]}`);
          } else if (type === 'CTO Used' && (parseFloat(row[7]) || 0) > 0) {
            // Fallback for simple CTO Used entries without detailed FIFO remarks (less accurate)
            // We need to guess which detail record it came from - this part is tricky without proper remarks
            // For now, let's log a warning if we can't find specific FIFO details
            Logger.log(`Warning: Found CTO Used entry for ${referenceId} but couldn't parse specific FIFO details from remarks: "${remarks}"`);
            // We could try to add the total CTO Used hours here, but reversing it accurately is hard.
            // It's better if the original deduction logged the source record ID in remarks.
          }
       }
    }
  }

  // --- MODIFICATION: Reverse the deductions array so we restore in the reverse order they were taken ---
  originalDeductions.reverse();
  // --- END MODIFICATION ---

  let totalRestored = 0;
  let remainingToRestore = hoursToRestore;
  const detailData = detailSheet.getDataRange().getValues();
  const updatesToDetailSheet = []; // Batch updates

  // Iterate through the identified original deductions
  for (const deduction of originalDeductions) {
    if (remainingToRestore <= 0) break;

    const hoursToRestoreHere = Math.min(deduction.hours, remainingToRestore);
    Logger.log(`Processing deduction: Need to restore ${hoursToRestoreHere} hrs originally from ${deduction.recordId}`);

    // Find the corresponding detail entry using the Record ID
    let detailRowIndex = -1;
    for (let i = 1; i < detailData.length; i++) {
        // --- MODIFICATION: Match using Record ID (column D, index 3) ---
      if (detailData[i][3] === deduction.recordId && detailData[i][1] === employeeId) {
        detailRowIndex = i + 1; // 1-based index for getRange
         Logger.log(`Found matching detail entry at row ${detailRowIndex}`);
        break;
      }
       // --- END MODIFICATION ---
    }

    if (detailRowIndex !== -1) {
      const currentRemaining = parseFloat(detailSheet.getRange(detailRowIndex, 7).getValue()) || 0; // Column G
      const newRemaining = currentRemaining + hoursToRestoreHere;
      const currentStatus = detailSheet.getRange(detailRowIndex, 9).getValue(); // Column I

      updatesToDetailSheet.push({ row: detailRowIndex, col: 7, value: newRemaining }); // Update Hours Remaining
       Logger.log(`Updating row ${detailRowIndex}, col 7 (Hours Remaining) from ${currentRemaining} to ${newRemaining}`);

      // If the entry was Depleted or Used, mark it back to Active
      if (currentStatus === 'Depleted' || currentStatus === 'Used') {
        updatesToDetailSheet.push({ row: detailRowIndex, col: 9, value: 'Active' }); // Update Status
         Logger.log(`Updating row ${detailRowIndex}, col 9 (Status) to Active`);
      }
      
      // Add a note about the restoration
       const currentNote = detailSheet.getRange(detailRowIndex, 11).getValue(); // Column K
       const TIME_ZONE = getScriptTimeZone();
       const newNote = currentNote + '\n[' + Utilities.formatDate(new Date(), TIME_ZONE, 'yyyy-MM-dd HH:mm') +
                      '] Restored ' + hoursToRestoreHere.toFixed(2) + ' hrs due to cancellation of ' + referenceId;
       updatesToDetailSheet.push({ row: detailRowIndex, col: 11, value: newNote});
       Logger.log(`Updating row ${detailRowIndex}, col 11 (Notes)`);


      totalRestored += hoursToRestoreHere;
      remainingToRestore -= hoursToRestoreHere;
    } else {
        Logger.log(`Warning: Could not find detail entry for Record ID ${deduction.recordId} to restore hours.`);
    }
  }

  // Apply batch updates to the detail sheet
  updatesToDetailSheet.forEach(update => {
    detailSheet.getRange(update.row, update.col).setValue(update.value);
  });
   Logger.log(`Applied ${updatesToDetailSheet.length} updates to COC_Balance_Detail.`);


  if (remainingToRestore > 0.01) { // Allow for small floating point inaccuracies
     Logger.log(`Warning: Could only restore ${totalRestored.toFixed(2)} out of ${hoursToRestore.toFixed(2)} hours requested. There might be discrepancies in ledger remarks or data.`);
     // Optionally throw an error here if exact restoration is critical
     // throw new Error(`Could only restore ${totalRestored.toFixed(2)} out of ${hoursToRestore.toFixed(2)} hours.`);
  }

  Logger.log(`Total hours restored: ${totalRestored.toFixed(2)}`);
  return totalRestored; // Return the actual amount restored
}


/**
 * Check and mark expired COC entries
 * Run this daily or on-demand
 */
function checkAndExpireCOC() {
  const detailSheet = ensureCOCBalanceDetailSheet();
  const data = detailSheet.getDataRange().getValues();
  const today = new Date();
  today.setHours(0, 0, 0, 0);

  let expiredCount = 0;
  let totalHoursForfeited = 0;
  const TIME_ZONE = getScriptTimeZone();

  for (let i = 1; i < data.length; i++) {
    const row = data[i];
    const status = String(row[DETAIL_COLS.STATUS] || '').trim();
    const hoursRemaining = parseFloat(row[DETAIL_COLS.HOURS_REMAINING]) || 0;
    const expirationVal = row[DETAIL_COLS.EXPIRATION_DATE];
    if (!expirationVal) continue;

    const expirationDate = new Date(expirationVal);
    if (isNaN(expirationDate.getTime())) continue;
    expirationDate.setHours(0, 0, 0, 0);

    if (status === 'Active' && hoursRemaining > 0 && expirationDate < today) {
      detailSheet.getRange(i + 1, DETAIL_COLS.STATUS + 1).setValue('Expired');

      const note = `[${Utilities.formatDate(new Date(), TIME_ZONE, 'yyyy-MM-dd HH:mm')}] EXPIRED - ${hoursRemaining.toFixed(2)} hours forfeited`;
      const currentNote = detailSheet.getRange(i + 1, DETAIL_COLS.REMARKS + 1).getValue() || '';
      const updatedNote = currentNote ? currentNote + '\n' + note : note;
      detailSheet.getRange(i + 1, DETAIL_COLS.REMARKS + 1).setValue(updatedNote);

      expiredCount++;
      totalHoursForfeited += hoursRemaining;

      const ledgerSheet = getDatabase().getSheetByName('COC_Ledger');
      if (ledgerSheet) {
        ledgerSheet.appendRow([
          generateLedgerId(),
          row[DETAIL_COLS.EMPLOYEE_ID],
          row[DETAIL_COLS.EMPLOYEE_NAME],
          new Date(),
          'COC Expired',
          row[DETAIL_COLS.RECORD_ID],
          0,
          hoursRemaining,
          '',
          row[DETAIL_COLS.MONTH_YEAR],
          expirationDate,
          Session.getActiveUser().getEmail(),
          'Auto-expired: ' + hoursRemaining.toFixed(2) + ' hours forfeited from ' + row[DETAIL_COLS.RECORD_ID]
        ]);
      }
    }
  }

  return {
    expiredCount: expiredCount,
    totalHoursForfeited: totalHoursForfeited.toFixed(2)
  };
}


/**
 * Iterates through all COC records and marks entries as expired when their
 * expiration date has passed. This function should be scheduled as a
 * time‑driven trigger (e.g. daily at midnight) so that expired entries are
 * automatically updated without user intervention.
 *
 * @return {number} The number of entries marked as expired.
 */
function checkExpiredCOC() {
  const db = getDatabase();
  const recSheet = db.getSheetByName('COC_Records');
  if (!recSheet) return 0;
  const data = recSheet.getDataRange().getValues();
  const today = new Date();
  let count = 0;
  for (let i = 1; i < data.length; i++) {
    const status = data[i][15];
    const expDate = data[i][14];
    if (status === 'Active' && expDate) {
      const exp = new Date(expDate);
      if (exp.getTime() <= today.getTime()) {
        recSheet.getRange(i + 1, 16).setValue('Expired');
        count++;
      }
    }
  }
  return count;
}


/**
 * Check if adding COC hours would exceed the monthly 40-hour limit
 */
function checkMonthlyLimitForMonth(employeeId, monthYear, hoursToAdd) {
  const db = getDatabase();
  const detailSheet = db.getSheetByName('COC_Balance_Detail');
  const TIME_ZONE = getScriptTimeZone();
  const MONTHLY_LIMIT = 40;

  if (!detailSheet || detailSheet.getLastRow() < 2) {
    return {
      valid: true,
      currentMonthTotal: 0,
      message: 'OK'
    };
  }

  const data = detailSheet.getDataRange().getValues();
  let monthTotal = 0;

  // Sum all COC earned in the same month for this employee
  for (let i = 1; i < data.length; i++) {
    const row = data[i];
    const empId = row[1]; // Employee ID
    const dateEarned = new Date(row[4]); // Date Earned
    const hoursEarned = row[5]; // Hours Earned

    if (empId === employeeId) {
      // Use MM-yyyy format to match the monthYear parameter
      const earnedMonthYear = Utilities.formatDate(dateEarned, TIME_ZONE, 'MM-yyyy');
      if (earnedMonthYear === monthYear) {
        monthTotal += hoursEarned;
      }
    }
  }

  const newTotal = monthTotal + hoursToAdd;

  if (newTotal > MONTHLY_LIMIT) {
    return {
      valid: false,
      currentMonthTotal: monthTotal,
      message: `Monthly limit exceeded! Current: ${monthTotal.toFixed(2)} hrs + New: ${hoursToAdd.toFixed(2)} hrs = ${newTotal.toFixed(2)} hrs. Maximum: ${MONTHLY_LIMIT} hrs/month`
    };
  }

  return {
    valid: true,
    currentMonthTotal: monthTotal,
    message: `OK. Monthly total: ${newTotal.toFixed(2)} hrs / ${MONTHLY_LIMIT} hrs`
  };
}


/**
 * Check if adding COC hours would exceed the 120-hour total balance limit
 */
function checkTotalBalanceLimitForEmployee(employeeId, hoursToAdd) {
  const BALANCE_LIMIT = 120;
  const currentBalance = getCurrentCOCBalanceFromDetail(employeeId);
  const newBalance = currentBalance + hoursToAdd;

  if (newBalance > BALANCE_LIMIT) {
    return {
      valid: false,
      currentBalance: currentBalance,
      message: `Total balance limit exceeded! Current: ${currentBalance.toFixed(2)} hrs + New: ${hoursToAdd.toFixed(2)} hrs = ${newBalance.toFixed(2)} hrs. Maximum: ${BALANCE_LIMIT} hrs`
    };
  }

  return {
    valid: true,
    currentBalance: currentBalance,
    message: `OK. Total balance: ${newBalance.toFixed(2)} hrs / ${BALANCE_LIMIT} hrs`
  };
}


// ============================================================================
// HELPER: Validate CTO Update Data (Separated Logic)
// ============================================================================
/**
 * Validates the data for updating a CTO application.
 * Returns null if valid, or an error message string if invalid.
 */
function validateCTOUpdate(application, newHours, newStartDateStr, newEndDateStr) {
     // 1. Check for valid hours (positive number, multiple of 4)
    if (isNaN(newHours) || newHours <= 0 || newHours % 4 !== 0) {
        return 'CTO hours must be a positive multiple of 4 (e.g., 4, 8, 12).';
    }

    // 2. Check for dates
    if (!newStartDateStr || !newEndDateStr) {
        return 'Start and End dates are required.';
    }

    const startDate = new Date(newStartDateStr + 'T00:00:00');
    const endDate = new Date(newEndDateStr + 'T00:00:00');

    // Check if dates are valid
     if (isNaN(startDate.getTime()) || isNaN(endDate.getTime())) {
        return 'Invalid date format provided.';
    }

    // 3. Check chronological order
    if (endDate < startDate) {
        return 'End date must not be before the start date.';
    }

     // 4. Check 5-day limit (inclusive)
    const diffDays = Math.floor((endDate.getTime() - startDate.getTime()) / (1000 * 60 * 60 * 24)) + 1;
    if (diffDays > 5) {
        return 'Date range must not exceed 5 days.';
    }

    // 5. Check 4/8 hour rule
    if ((newHours === 4 || newHours === 8) && startDate.getTime() !== endDate.getTime()) {
        return 'For 4 or 8-hour CTO applications, the start and end dates must be the same.';
    }

     // 6. Check balance IF hours are increasing AND status is Approved
     if (application.status === 'Approved' && newHours > application.oldHours) {
        const currentBalance = getCurrentCOCBalance(application.employeeId);
        const additionalHoursNeeded = newHours - application.oldHours;
        if (currentBalance < additionalHoursNeeded) {
            return `Insufficient COC balance. Employee has ${currentBalance.toFixed(2)} hrs available, but needs ${additionalHoursNeeded.toFixed(2)} additional hours for this update.`;
        }
    }

    // All checks passed
    return null;
}


/**
 * Check FIFO integrity for a specific employee
 * @param {string} employeeId Employee ID to check
 * @param {Array} detailData COC_Balance_Detail data
 * @param {Array} ctoData CTO_Applications data
 * @return {Array} Array of issues found
 */
function checkEmployeeFIFO(employeeId, detailData, ctoData) {
  const issues = [];
  
  // Get all COC entries for this employee, sorted by certificate date (FIFO order)
  const cocEntries = [];
  for (let i = 1; i < detailData.length; i++) {
    const row = detailData[i];
    if (row[DETAIL_COLS.EMPLOYEE_ID] === employeeId && String(row[DETAIL_COLS.STATUS] || '').trim() === 'Active') {
      let certificateDate = row[DETAIL_COLS.CERTIFICATE_DATE];
      if (!(certificateDate instanceof Date) || isNaN(certificateDate.getTime())) {
        certificateDate = parseMonthYear(row[DETAIL_COLS.MONTH_YEAR]) || new Date(row[DETAIL_COLS.DATE_CREATED]);
      }

      let expirationDate = row[DETAIL_COLS.EXPIRATION_DATE];
      if (!(expirationDate instanceof Date) || isNaN(expirationDate.getTime())) {
        expirationDate = null;
      }

      cocEntries.push({
        rowNumber: i + 1,
        recordId: row[DETAIL_COLS.RECORD_ID],
        certificateDate: certificateDate,
        hoursEarned: parseFloat(row[DETAIL_COLS.HOURS_EARNED]) || 0,
        hoursUsed: parseFloat(row[DETAIL_COLS.HOURS_USED]) || 0,
        hoursRemaining: parseFloat(row[DETAIL_COLS.HOURS_REMAINING]) || 0,
        expirationDate: expirationDate
      });
    }
  }

  // Sort by certificate date (oldest first - FIFO)
  cocEntries.sort((a, b) => a.certificateDate - b.certificateDate);

  // Get all approved CTO applications for this employee
  const ctoApplications = [];
  for (let i = 1; i < ctoData.length; i++) {
    const row = ctoData[i];
    if (row[1] === employeeId && row[8] === 'Approved') { // Column B: Employee ID, Column I: Status
      ctoApplications.push({
        applicationId: row[0],
        applicationDate: new Date(row[2]),
        hoursRequested: parseFloat(row[3]) || 0
      });
    }
  }

  // Sort CTO applications by date
  ctoApplications.sort((a, b) => a.applicationDate - b.applicationDate);

  // Simulate FIFO deduction and compare with actual
  let simulatedDeductions = cocEntries.map(entry => ({
    recordId: entry.recordId,
    expectedUsed: 0,
    expectedRemaining: entry.hoursEarned
  }));

  for (const cto of ctoApplications) {
    let remainingToDeduct = cto.hoursRequested;
    
    for (let i = 0; i < simulatedDeductions.length && remainingToDeduct > 0.01; i++) {
      const availableHours = simulatedDeductions[i].expectedRemaining;
      
      if (availableHours > 0) {
        const hoursToDeduct = Math.min(availableHours, remainingToDeduct);
        simulatedDeductions[i].expectedUsed += hoursToDeduct;
        simulatedDeductions[i].expectedRemaining -= hoursToDeduct;
        remainingToDeduct -= hoursToDeduct;
      }
    }
    
    if (remainingToDeduct > 0.01) {
      issues.push({
        type: 'INSUFFICIENT_BALANCE',
        ctoApplicationId: cto.applicationId,
        message: `CTO application ${cto.applicationId} deducted hours when insufficient balance existed`,
        shortfall: remainingToDeduct.toFixed(2)
      });
    }
  }

  // Compare simulated vs actual
  for (let i = 0; i < cocEntries.length; i++) {
    const actual = cocEntries[i];
    const simulated = simulatedDeductions.find(s => s.recordId === actual.recordId);
    
    if (simulated) {
      const usedDiff = Math.abs(actual.hoursUsed - simulated.expectedUsed);
      const remainingDiff = Math.abs(actual.hoursRemaining - simulated.expectedRemaining);
      
      if (usedDiff > 0.01 || remainingDiff > 0.01) {
        issues.push({
          type: 'FIFO_VIOLATION',
          recordId: actual.recordId,
          certificateDate: actual.certificateDate,
          actualUsed: actual.hoursUsed.toFixed(2),
          expectedUsed: simulated.expectedUsed.toFixed(2),
          actualRemaining: actual.hoursRemaining.toFixed(2),
          expectedRemaining: simulated.expectedRemaining.toFixed(2),
          message: 'Hours used/remaining do not match FIFO order'
        });
      }
    }
  }

  // Check for negative balances
  for (const entry of cocEntries) {
    if (entry.hoursRemaining < -0.01) {
      issues.push({
        type: 'NEGATIVE_BALANCE',
        recordId: entry.recordId,
        hoursRemaining: entry.hoursRemaining.toFixed(2),
        message: 'COC entry has negative remaining hours'
      });
    }
  }

  return issues;
}


/**
 * Calculates overtime hours, the appropriate multiplier, and the resulting COC
 * earned for a single date. The day type is automatically determined by
 * checking the day of week and any matching entry in the Holidays sheet.
 * Weekdays allow overtime only between 5:00 PM and 7:00 PM (maximum 2 hours).
 * Weekends and holidays allow morning (8:00–12:00) and afternoon (1:00–5:00)
 * blocks, excluding the lunch break from 12:01–12:59 PM. For weekends and
 * holidays the multiplier is 1.5. All calculations are capped as per the
 * business rules.
 *
 * @param {Date} date The calendar date being processed.
 * @param {string} amIn  Optional AM start time in HH:MM (24‑hour) format.
 * @param {string} amOut Optional AM end time in HH:MM (24‑hour) format.
 * @param {string} pmIn  Optional PM start time in HH:MM (24‑hour) format.
 * @param {string} pmOut Optional PM end time in HH:MM (24‑hour) format.
 * @return {Object} An object describing the day type, hours worked, multiplier
 * and COC earned.
*/
function calculateOvertimeForDate(date, amIn, amOut, pmIn, pmOut) {
  // Get settings from the 'Settings' sheet (assuming getSettings exists in HelperFunctions.gs)
  const settings = getSettings();
  // Get the script's timezone (assuming getScriptTimeZone exists in HelperFunctions.gs)
  const TIME_ZONE = getScriptTimeZone();

  // Determine the day type and base multiplier using the enhanced function
  // (assuming getDayTypeEnhanced exists in HelperFunctions.gs)
  const dayInfo = getDayTypeEnhanced(date);
  const dayType = dayInfo.dayType;
  const multiplier = dayInfo.multiplier; // This will be 1.0 for Weekday, 1.5 for Weekend/Holiday

  let hoursWorked = 0; // Initialize total hours worked

  /**
   * Helper function to convert a time string (HH:mm) to the total number of minutes from midnight.
   * Returns null if the input string is invalid or empty.
   * @param {string} timeStr Time string in HH:mm format.
   * @return {number|null} Total minutes from midnight or null.
   */
  function timeToMinutes(timeStr) {
    if (!timeStr) return null; // Return null for empty or null input
    const parts = timeStr.split(':');
    // Ensure exactly two parts and both are numbers
    if (parts.length !== 2 || isNaN(parts[0]) || isNaN(parts[1])) return null;
    const h = parseInt(parts[0], 10);
    const m = parseInt(parts[1], 10);
    // Basic validation for hours and minutes range
    if (h < 0 || h > 23 || m < 0 || m > 59) return null;
    return h * 60 + m; // Calculate total minutes
  }

  // Convert provided times to minutes
  const amInMins = timeToMinutes(amIn);
  const amOutMins = timeToMinutes(amOut);
  const pmInMins = timeToMinutes(pmIn);
  const pmOutMins = timeToMinutes(pmOut);

  // --- Overtime Calculation Logic ---

  // Check if it's a Weekday
  if (dayType === 'Weekday') {
    // Define the valid overtime window for weekdays (5 PM to 7 PM)
    const weekdayStartMins = 17 * 60; // 5:00 PM
    const weekdayEndMins = 19 * 60;   // 7:00 PM

    let weekdayOvertimeMins = 0;

    // Calculate overlap within the 5 PM - 7 PM window for AM times (unlikely but possible if data entry allows)
    if (amInMins !== null && amOutMins !== null && amOutMins > amInMins) {
      const start = Math.max(amInMins, weekdayStartMins); // Find the later start time
      const end = Math.min(amOutMins, weekdayEndMins);   // Find the earlier end time
      if (end > start) { // Check if there is an overlap
        weekdayOvertimeMins += (end - start);
      }
    }

    // Calculate overlap within the 5 PM - 7 PM window for PM times
    if (pmInMins !== null && pmOutMins !== null && pmOutMins > pmInMins) {
      const start = Math.max(pmInMins, weekdayStartMins); // Find the later start time
      const end = Math.min(pmOutMins, weekdayEndMins);   // Find the earlier end time
      if (end > start) { // Check if there is an overlap
        weekdayOvertimeMins += (end - start);
      }
    }

    // Convert total valid overtime minutes to hours
    hoursWorked = weekdayOvertimeMins / 60.0;
    // Ensure weekday overtime doesn't exceed 2 hours (double-check, though window restricts this)
    hoursWorked = Math.min(hoursWorked, 2.0);

  } else { // Calculation for Weekends and Holidays (multiplier is already 1.5)
    // Define standard work block times and lunch break
    const amBlockStartMins = 8 * 60; // 8:00 AM
    const amBlockEndMins = 12 * 60;  // 12:00 PM
    const pmBlockStartMins = 13 * 60; // 1:00 PM
    const pmBlockEndMins = 17 * 60;  // 5:00 PM

    let nonWeekdayOvertimeMins = 0;

    // Calculate overlap with AM block (8 AM - 12 PM)
    if (amInMins !== null && amOutMins !== null && amOutMins > amInMins) {
      const start = Math.max(amInMins, amBlockStartMins);
      const end = Math.min(amOutMins, amBlockEndMins);
      if (end > start) {
        nonWeekdayOvertimeMins += (end - start);
      }
    }

    // Calculate overlap with PM block (1 PM - 5 PM)
    if (pmInMins !== null && pmOutMins !== null && pmOutMins > pmInMins) {
      const start = Math.max(pmInMins, pmBlockStartMins);
      const end = Math.min(pmOutMins, pmBlockEndMins);
      if (end > start) {
        nonWeekdayOvertimeMins += (end - start);
      }
    }

    // Convert total valid overtime minutes to hours
    hoursWorked = nonWeekdayOvertimeMins / 60.0;
  }

  // --- End Overtime Calculation Logic ---

  // Final calculation of COC earned: total valid hours worked * multiplier
  // Ensure hoursWorked is not negative due to any potential logic edge cases
  hoursWorked = Math.max(0, hoursWorked);
  const cocEarned = hoursWorked * multiplier;

  // Return the results including the original times for reference
  return {
    dayType: dayType,           // e.g., "Weekday", "Weekend", "Regular Holiday"
    hoursWorked: hoursWorked,   // Calculated valid overtime hours
    multiplier: multiplier,     // Multiplier used (1.0 or 1.5)
    cocEarned: cocEarned,       // Resulting COC hours earned
    amIn: amIn || '',           // Original AM In time passed to function
    amOut: amOut || '',         // Original AM Out time passed to function
    pmIn: pmIn || '',           // Original PM In time passed to function
    pmOut: pmOut || ''          // Original PM Out time passed to function
  };
}

/**
 * Calculate COC for a specific date with enhanced holiday logic
 * @param {Date} date The date to calculate COC for
 * @param {number} hoursWorked The hours worked
 * @param {string} timeIn Optional time in (for half-day calculations)
 * @param {string} timeOut Optional time out (for half-day calculations)
 * @return {Object} Object with COC hours and day type info
 */
function calculateCOCForDate(date, hoursWorked, timeIn, timeOut) {
  const dayInfo = getDayTypeEnhanced(date);
  let cocHours = 0;
  
  // For half-day holidays, we need to calculate based on time worked
  if (dayInfo.dayType === 'Half-day Holiday' && dayInfo.additionalInfo.halfdayTime && timeIn && timeOut) {
    // Parse times
    const halfdayStartMins = timeToMinutes(dayInfo.additionalInfo.halfdayTime);
    const timeInMins = timeToMinutes(timeIn);
    const timeOutMins = timeToMinutes(timeOut);
    
    // Calculate hours before and after half-day start
    if (timeOutMins <= halfdayStartMins) {
      // All work was before half-day, use normal weekday rate
      cocHours = hoursWorked * 1.0;
    } else if (timeInMins >= halfdayStartMins) {
      // All work was during half-day, use holiday rate
      cocHours = hoursWorked * 1.5;
    } else {
      // Work spanned both periods, calculate proportionally
      const hoursBeforeHalfday = (halfdayStartMins - timeInMins) / 60;
      const hoursAfterHalfday = (timeOutMins - halfdayStartMins) / 60;
      cocHours = (hoursBeforeHalfday * 1.0) + (hoursAfterHalfday * 1.5);
    }
  } else {
    // Standard calculation using multiplier
    cocHours = hoursWorked * dayInfo.multiplier;
  }
  
  return {
    cocHours: cocHours,
    dayType: dayInfo.dayType,
    multiplier: dayInfo.multiplier,
    additionalInfo: dayInfo.additionalInfo
  };
}

