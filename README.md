# Radsystems-Portal
/****************************************************
 * RadSystems — Enhanced Assignment System
 * Features:
 * 1. Heads can upload files when creating assignments
 * 2. Students can download/upload assignment files
 * 3. Assignment counter decreases when students resubmit
 * 4. Tracked upload history
 ****************************************************/

// ... [Previous CONFIG, SHEETS, HEADERS remain the same] ...

/** MODIFIED ASSIGNMENT CREATION **/
function api_createAssignment(token, { title, course, unit, deadline, file }) {
  const user = validateToken(token);
  if (!user || user.role !== 'head') throw new Error('Unauthorized');
  
  // Create assignment first
  const assignmentId = uuid();
  const assignmentData = {
    assignmentId,
    title: title || '',
    course,
    unit: unit || '',
    createdBy: user.userId,
    deadline: deadline || '',
    isActive: true,
    fileUrl: '', // Will be set after upload
    totalStudents: 0, // Will be updated
    remainingSubmissions: 0 // Will be updated
  };

  // Handle file upload if present
  if (file && file.base64) {
    const folder = ensureUploadsFolder(course, 'assignments');
    const blob = Utilities.newBlob(
      Utilities.base64Decode(file.base64),
      file.mimeType,
      file.filename
    );
    const uploadedFile = folder.createFile(blob);
    assignmentData.fileUrl = uploadedFile.getUrl();
  }

  // Write to sheet
  writeObject(getSheet(SHEETS.ASSIGNMENTS), assignmentData);
  
  // Count students for this course
  const students = readObjects(getSheet(SHEETS.STUDENTS))
    .filter(s => s.course === course && s.status === 'Active');
  
  assignmentData.totalStudents = students.length;
  assignmentData.remainingSubmissions = students.length;
  
  // Update the counts
  updateFirst(getSheet(SHEETS.ASSIGNMENTS),
    a => a.assignmentId === assignmentId,
    {
      totalStudents: students.length,
      remainingSubmissions: students.length
    }
  );

  return { 
    ok: true,
    assignment: assignmentData
  };
}

/** ENHANCED STUDENT DASHBOARD **/
function api_getStudentDashboard(token) {
  const user = validateToken(token);
  if (!user || user.role !== 'student') throw new Error('Unauthorized');
  
  const student = readObjects(getSheet(SHEETS.STUDENTS))
    .find(s => s.studentId === user.userId);
  if (!student) throw new Error('Student not found');

  const assignments = readObjects(getSheet(SHEETS.ASSIGNMENTS))
    .filter(a => a.course === student.course && a.isActive)
    .map(a => ({
      ...a,
      submitted: false, // Will be updated below
      canSubmit: new Date(a.deadline) > new Date() // Before deadline
    }));

  const checks = readObjects(getSheet(SHEETS.CHECKS))
    .filter(c => c.studentId === user.userId);

  // Mark submitted assignments
  assignments.forEach(a => {
    a.submitted = checks.some(c => c.assignmentId === a.assignmentId);
  });

  return {
    ok: true,
    student,
    assignments,
    stats: {
      total: assignments.length,
      submitted: assignments.filter(a => a.submitted).length,
      pending: assignments.filter(a => !a.submitted).length
    }
  };
}

/** ENHANCED SUBMIT CHECK **/
function api_submitCheck(token, { assignmentId, studentId, status, grade, comment, file }) {
  const user = validateToken(token);
  if (!user) throw new Error('Unauthorized');

  // Verify student is submitting their own work
  if (user.role === 'student' && user.userId !== studentId) {
    throw new Error('You can only submit your own work');
  }

  const now = nowISO();
  let fileUrl = '';

  // Handle file upload
  if (file && file.base64) {
    const folder = ensureUploadsFolder(
      user.course, 
      'submissions', 
      assignmentId, 
      studentId
    );
    const blob = Utilities.newBlob(
      Utilities.base64Decode(file.base64),
      file.mimeType,
      file.filename
    );
    const uploadedFile = folder.createFile(blob);
    fileUrl = uploadedFile.getUrl();
  }

  const checksSheet = getSheet(SHEETS.CHECKS);
  const existing = readObjects(checksSheet)
    .find(c => c.assignmentId === assignmentId && c.studentId === studentId);

  // Update remaining submissions counter if this is a new submission
  if (!existing && user.role === 'student') {
    const assignmentsSheet = getSheet(SHEETS.ASSIGNMENTS);
    const assignment = readObjects(assignmentsSheet)
      .find(a => a.assignmentId === assignmentId);
    
    if (assignment) {
      const newCount = Math.max(0, (assignment.remainingSubmissions || 0) - 1);
      updateFirst(assignmentsSheet,
        a => a.assignmentId === assignmentId,
        { remainingSubmissions: newCount }
      );
    }
  }

  // Prepare update
  const update = {
    assignmentId,
    studentId,
    assistantId: user.role === 'assistant' ? user.userId : existing?.assistantId || '',
    status: status || existing?.status || '',
    grade: asText(grade || existing?.grade || ''),
    comment: comment || existing?.comment || '',
    fileUrl: fileUrl || existing?.fileUrl || '',
    updatedAt: now,
    submissionCount: (existing?.submissionCount || 0) + 1
  };

  // Add edit notice if modified by admin/head
  if (existing && (user.role === 'head' || user.role === 'assistant')) {
    update.comment = (existing.comment ? existing.comment + '\n\n' : '') + 
      `[${now}] Edited by ${user.displayName}`;
  }

  // Update or create record
  if (existing) {
    updateFirst(checksSheet, 
      c => c.assignmentId === assignmentId && c.studentId === studentId,
      update
    );
  } else {
    writeObject(checksSheet, {
      checkId: uuid(),
      ...update,
      createdAt: now
    });
  }

  return { ok: true };
}

/** ENHANCED ASSIGNMENTS LIST **/
function api_getHeadDashboard(token) {
  const user = validateToken(token);
  if (!user || user.role !== 'head') throw new Error('Unauthorized');
  
  const assignments = readObjects(getSheet(SHEETS.ASSIGNMENTS))
    .filter(a => a.course === user.course)
    .map(a => ({
      ...a,
      completion: a.totalStudents > 0 
        ? Math.round(100 * (a.totalStudents - a.remainingSubmissions) / a.totalStudents)
        : 0
    }));

  return {
    ok: true,
    assignments,
    stats: {
      total: assignments.length,
      active: assignments.filter(a => a.isActive).length,
      avgCompletion: assignments.length > 0
        ? Math.round(assignments.reduce((sum, a) => sum + a.completion, 0) / assignments.length)
        : 0
    }
  };
}
/**
 * Required for web app deployment (even if unused)
 */
function doGet(e) {
  return ContentService.createTextOutput(
    JSON.stringify({ 
      status: "running", 
      note: "This is a backend API. Use POST requests." 
    })
  ).setMimeType(ContentService.MimeType.JSON);
}
// ... [Keep all other existing functions] ...

