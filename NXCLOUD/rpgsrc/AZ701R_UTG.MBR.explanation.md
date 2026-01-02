# RPG Program AZ701R - Explanation

This ILE RPG program, named **AZ701R**, is designed to **unzip (extract) files from a ZIP archive** using Java classes via RPG's Java integration capabilities. Here’s a breakdown of how the program works and how it's structured:

---

## 1. **Header and Program Description**

The header provides basic details:
- **Description:** "Zipper ut en fil" (Norwegian, translates to "Unzips a file").
- **Creation Date:** 25.02.2003 by "MHA".

---

## 2. **Data Definitions**

The program uses a mix of RPG data types and Java object references. Below are key variable definitions:

### **Java Object References**
```rpg
d zipfil          s               o   class(*JAVA:'java.lang.String')
d directory       s               o   class(*JAVA:'java.lang.String')
d logfil          s               o   class(*JAVA:'java.lang.String')
d uz              s               o   class(*JAVA:'asp.zip.UnpackZip')
d antall          s             10i 0
```
- **zipfil, directory, logfil:** RPG variables that hold Java String objects.
- **uz:** Java object of type `asp.zip.UnpackZip` – likely a custom Java class for unzipping files.
- **antall:** Standard RPG integer (10,0); receives the count of files unzipped.

### **Input Parameters**
```rpg
d p_zip           s             50a
d p_log           s             50a
d p_dir           s             50a
d p_ant           s                   like(antall)
```
- **p_zip:** ZIP file name/path.
- **p_log:** Log file name/path.
- **p_dir:** Directory where files should be extracted.
- **p_ant:** Returned count of files extracted.

---

## 3. **Java Constructors and Method Prototypes**

These prototypes define RPG calls into Java:

- **mkstr:** Calls Java's `String` constructor to make a Java string from an RPG string.
- **crtzipobj:** Instantiates the `asp.zip.UnpackZip` Java object.
- **setzip, setlog, setdir:** Call Java methods on the unzip object to set the ZIP file, log file, and directory.
- **dezip:** Calls the Java method to do the actual unzipping and returns the count of files unzipped.

---

## 4. **Main Program Logic**

### **Object Creation**
```rpg
c                   eval      uz = crtzipobj
```
- Instantiates the unzip object (`uz`).

### **Assign and Pass Parameters**
#### **ZIP File**
```rpg
c                   eval      zipfil = mkstr(%trim(p_zip))
c                   callp     setzip(uz:zipfil)
```
- Converts the input ZIP file path/string into a Java String object and passes it to the unzip object.

#### **Log File (Optional)**
```rpg
c                   if        p_log <> *blank and
c                             %trim(p_log) <> '*NONE' and
c                             %trim(p_log) <> '*BLANK'
c                   eval      logfil = mkstr(%trim(p_log))
c                   callp     setlog(uz:logfil)
c                   endif
```
- If the log file parameter is provided (not blank, "*NONE", or "*BLANK"), it is converted and set in the Java object.

#### **Output Directory (Optional)**
```rpg
c                   if        p_dir <> *blank and
c                             %trim(p_dir) <> '*NONE' and
c                             %trim(p_dir) <> '*BLANK'
c                   eval      directory = mkstr(%trim(p_dir))
c                   callp     setdir(uz:directory)
c                   endif
```
- If the directory parameter is provided, it is converted and set.

### **Perform Unzip and Capture Result**
```rpg
c                   eval      antall = dezip(uz)
```
- Performs the actual extraction by calling the Java method, and saves number of files extracted into `antall`.

### **Output the Result**
```rpg
c                   eval      p_ant = antall
```
- Stores the result (number of files unpacked) into the output parameter.

### **Program End**
```rpg
c                   eval      *inlr = *on
c                   return
```
- Sets the last record indicator, causing the program to end and cleanup.

---

## 5. **Program Initialization Subroutine**

The `*inzsr` subroutine is called automatically on program start:

```rpg
c     *inzsr        begsr
c     *entry        plist
c                   parm                    p_zip
c                   parm                    p_dir
c                   parm                    p_log
c                   parm                    p_ant
c                   endsr
```
Defines program parameters:
- **p_zip:** ZIP file path.
- **p_dir:** Output directory.
- **p_log:** Log file.
- **p_ant:** Output parameter for number of files extracted.

---

## 6. **Overall Workflow**

1. **Program receives input parameters** (ZIP file, output dir, log file).
2. **Creates a Java unzip object** via RPG-Java integration.
3. **Passes parameters** to methods of the Java object.
4. **Calls the Java `unpackFiles` method** to do the extraction.
5. **Returns the result** (number of files unpacked).

---

## 7. **Special Points**

- **Java Integration:** Program relies on a Java class (`asp.zip.UnpackZip`) to do the unzipping. The RPG program acts as a wrapper or bridge.
- **Parameter Flexibility:** Input directory and log file are optional and validated before setting.
- **Return Value:** Number of files unzipped is returned to the caller via `p_ant`.

---

## 8. **Use Cases**

Typical use would be in a CL program, batch job, or another RPG program that needs to extract files from a ZIP archive in the IFS (Integrated File System) on IBM i.

---

## 9. **Key Takeaway**

**AZ701R** provides a simple interface for unzipping files on the IBM i platform by leveraging Java from RPG, handling parameter checking, and reporting output concisely for further processing.

---