<h2>Program Overview</h2>
<ul>
  <li><b>Name:</b> EDIJOB</li>
  <li><b>Purpose:</b> Read from the LWELPF file (renamed to LWELL2R), filter records by date/time and optional document number, and write selected lines to the LWEBPF output file (renamed to LWEBPFR).</li>
  <li><b>Date format:</b> DMY (day-month-year)</li>
</ul>

<h2>File Declarations</h2>
<ul>
  <li>Input file<br>
    <code>FLWELL2 IF E K DISK RENAME(LWELPFR:LWELL2R)</code><br>
    • Alias LWELL2R refers to the original LWELPF</li>
  <li>Output file<br>
    <code>FLWEBPF UF A E K DISK RENAME(LWEBPFR:LWEBPFR)</code><br>
    • Alias LWEBPFR refers to the original LWEBPF</li>
</ul>

<h2>Data Definitions</h2>
<ul>
  <li>UDS Local Data Area (LDA)<br>
      <code>l_firm</code> at positions 944–946 (unused in code)</li>
  <li>Key‐field variables for input file<br>
      <code>lwell2_logind</code>, <code>lwell2_logtyp</code>, <code>lwell2_logdat</code> defined <i>LIKE</i> their file counterparts</li>
  <li>Control variables</li>
  <ul>
    <li><code>w_dato</code> (6-digit date)</li>
    <li><code>w_time</code> (6-digit time filter)</li>
    <li><code>w_isotim</code> (time field from record)</li>
    <li><code>w_dokn</code> (10-char document number filter)</li>
    <li><code>w_rest</code> (1-char “rest” flag)</li>
    <li><code>w_recid</code> (4-char record ID, e.g. 'ED01')</li>
    <li><code>w_doktst</code> (10-char document number extracted from record)</li>
    <li><code>b_start</code> (1-char flag indicating when to begin writing)</li>
  </ul>
</ul>

<h2>Main Processing Logic</h2>
<ol>
  <li>
    Initialize:<br>
    <code>b_start = *OFF</code><br>
    Move <code>w_time</code> into <code>w_isotim</code> (for later comparisons).
  </li>
  <li>
    Position input file:<br>
    <code>SETLL L007 Key on LWELL2R</code> (uses the key list set up in <code>*INZSR</code>).
  </li>
  <li>
    Loop through input records:<br>
    <code>READE LWELL2R</code> until <code>%EOF</code>.
    <ul>
      <li>
        <b>Time filter:</b><br>
        If <code>w_time <> 0</code> and record’s <code>LOGTID</code> ≠ <code>w_isotim</code>, skip to next record.
      </li>
      <li>
        <b>Extract record ID and document number:</b><br>
        <code>w_recid = %SUBST(LOGREC:1:4)</code><br>
        <code>w_doktst = %SUBST(LOGREC:38:10)</code>
      </li>
      <li>
        <b>Start‐writing logic:</b><br>
        • If no document filter (<code>w_dokn</code> blank), set <code>b_start = *ON</code> immediately.<br>
        • Otherwise, wait until you find an ED01 record whose <code>w_doktst</code> matches <code>w_dokn</code>, then set <code>b_start = *ON</code>.
      </li>
      <li>
        <b>End‐of‐document group check:</b><br>
        If already started AND a document filter is in effect AND <code>w_rest <> 1</code> AND you hit another ED01 with a different <code>w_doktst</code>, exit the loop.
      </li>
      <li>
        <b>Write to output:</b><br>
        If <code>b_start = *ON</code>, move <code>LOGREC</code> into <code>BESLIN</code> and <code>WRITE LWEBPFR</code>.
      </li>
    </ul>
  </li>
  <li>End program:<br>
    <code>*INLR = *ON</code> to close files and return to caller.
  </li>
</ol>

<h2>Initialization Subroutine (*INZSR)</h2>
<ul>
  <li>Entry parameters (<code>PLIST</code>):<br>
      <code>w_dato</code>, <code>w_time</code>, <code>w_dokn</code>, <code>w_rest</code></li>
  <li>Define key list <code>LWELL2_KEY</code> containing:<br>
      <code>lwell2_logind</code>, <code>lwell2_logtyp</code>, <code>lwell2_logdat</code></li>
  <li>Populate key fields:<br>
      <code>lwell2_logind = 'I'</code><br>
      <code>lwell2_logtyp = 'ORDR'</code><br>
      <code>lwell2_logdat = w_dato</code></li>
  <li>Return to main logic.</li>
</ul>