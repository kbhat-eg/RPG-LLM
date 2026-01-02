# RPG Source Code Explanation: Modulus-11 Check Digit Calculation (ASMD11)

This ILE RPG program, likely written for a banking or business environment, calculates and verifies Modulus-11 (MOD11) check digits. The program is particularly customized for cases where the check digit may be a minus sign ("-"), which is non-standard but sometimes required in Scandinavian banking and payment systems.

Below is a breakdown and explanation of the code:

---

## **Header and Documentation**

The `h/TITLE` and comment block describe the program:

- **Program name**: ASMD11
- **Purpose**: Calculates and verifies Modulus-11 number, allowing for minus in the check digit.
- **Clients/Usage**: Customized for "Mestergruppen"
- **Developer notes**: Reference, change descriptions, etc.

---

## **Variable and Data Structure Definitions**

```rpg
d m11             s              1  0 dim(24)
d u11             s              2  0 dim(24)
```
- `m11`: Array for the numeric value of each digit (max 24).
- `u11`: Another array used for storing intermediate multiplications.

**Data Structures:**
```rpg
d                 ds
d  qdmd11                 1     25                                         
d  q11                    1     24                                         
d                                     dim(24)                              
d  qdck11                25     25  0                                      
A    d  qdck11A               25     25                                     
```
- `qdmd11`: 25-character input (the full number including check digit).
- `q11`: Array view of the first 24 digits/characters.
- `qdck11`: The check digit (25th character, as numeric).
- `qdck11A`: The check digit as alphanumeric (may be '-', '0', etc).

Other variables:
- `fa11`: Factor for multiplication, starts at 7 and decrements, cycles at 2.
- `ix`, `zz`: Index and loop counters.
- `ksifhj`: Temporary for check digit calculation.
- `modr11`, `mods11`: For MOD11 calculation results.
- `wplist`: Passed parameter, 25-char input number.

---

## **Main Routine**

### **Parameter List and Initialization**

```rpg
c     *entry        plist
c                   parm                    wplist
...
c                   eval      qdmd11 = wplist
```

- The program gets called with a 25-character value (`wplist`), which it copies to the internal data structure.

```rpg
c                   z-add     1             zz
c                   z-add     7             fa11
c                   z-add     0             mods11
c                   z-add     0             modr11
c                   z-add     0             ksifhj
c                   z-add     1             ix
```

- Initialize counters and accumulators.

---

### **Step 1: Prepare Input Array**

```rpg
B001 c                   dou       ix > 24
B002 c                   if        q11(ix) = *blank
 002 c                   move      *zero         m11(ix)
X002 c                   else
 002 c                   move      q11(ix)       m11(ix)
E002 c                   endif
 001 c                   add       1             ix
E001 c                   enddo
```
- For each character (1 to 24):
    - If blank, insert 0 into `m11`.
    - Otherwise, copy the numeric value.
- Prepares input for MOD11 calculation.

---

### **Step 2: Calculate Weighted Sum (Modulus-11 Algorithm)**

#### **Weighting Logic**

```rpg
c     utr11         tag
c     m11(zz)       mult      fa11          u11(zz)
```

- Multiply each digit by the weighting factor (`fa11`), store in `u11`.

#### **Weight Factor Decrement and Cycling**

```rpg
B001 c                   if        zz = 24
 001 c                   z-add     1             zz
 001 c                   goto      add11
X001 c                   else
 001 c                   add       1             zz
B002 c                   if        fa11 = 2
 002 c                   z-add     7             fa11
X002 c                   else
 002 c                   sub       1             fa11
E002 c                   endif
 001 c                   goto      utr11
E001 c                   endif
```

- For each digit:
    - Weighting factor (`fa11`) starts at 7, decrements by 1 each time.
    - When it reaches 2, resets to 7 (cycles 7,6,5,4,3,2,7,6,...).
    - Loops until all 24 digits processed.

---

### **Step 3: Calculate Weighted Sum**

```rpg
c     add11         tag
B001 c                   if        zz > 24
 001 c                   goto      sif11
E001 c                   endif
c                   add       u11(zz)       mods11
c                   add       1             zz
c                   goto      add11
```
- Sums all the products (`u11`) into `mods11`.

---

### **Step 4: Derive Check Digit**

```rpg
c     sif11         tag
c                   div       11            mods11
c                   mvr                     modr11
```
- Divide the sum by 11, store the remainder in `modr11`.

#### **Check Digit Assignment (Special Handling)**

```rpg
B001 c                   if        modr11 = 1
A001 c                   move      '-'           qdck11A
X001 c                   else
B002 c                   if        modr11 = 0
 002 c                   move      '0'           qdck11
X002 c                   else
 002 c     11            sub       modr11        ksifhj
 002 c                   eval      qdck11 = ksifhj
E002 c                   endif
E001 c                   endif
```

- **If remainder is 1**: check digit is a minus `'-'`.
- **If remainder is 0**: check digit is `'0'`.
- **Else**: check digit is `11 - remainder` (as per MOD11 standard).

---

### **Finish/Return**

```rpg
c     slutt         tag
c                   eval      wplist = qdmd11
c                   return
```
- Updates the passed input variable with the result.
- Returns to the calling program.

---

## **Summary**

- Input: A 25-character number, the last is the check digit.
- Output: The same structure, with a newly calculated/verified check digit.
- The program:
    - Strips blanks/pads zeros.
    - Multiplies digits by a cycling weight sequence (7 to 2, repeats).
    - Sums those products.
    - Calculates the remainder after dividing by 11.
    - Sets the check digit as per MOD11, with special support for a minus sign if the remainder is 1.

---

## **Use Cases**

- Verification and calculation of account numbers, payment references, or similar IDs that use MOD11 check digit logic.
- Customized for applications where a minus sign can be a valid check digit.

---

## **Key Points for Onboarding**

- Understand the MOD11 check digit logic.
- Weighting sequence cycles 7 → 2 and repeats.
- Handles up to 24 input digits (plus one check digit).
- The code uses traditional RPG cycle and array processing (not RPG IV/free format).
- Special handling for Scandinavian banking conventions (minus sign for check digit).

---

Feel free to ask about any specific part, or for examples of how to invoke the program!