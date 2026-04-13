# **CYBR 3050 – Lab: Privacy in Practice – Metadata & Personal Information Exposure**

---

## Background

In modern cybersecurity, protecting data is not just about securing files—it is also about understanding the **hidden data embedded within them**. Many common file types (images, PDFs, Office documents) contain **metadata** that can expose sensitive information such as:

* GPS coordinates
* Device information
* Author names
* Organizational details

Additionally, web browsers continuously expose information about users through **fingerprinting**, often without explicit consent.

In this lab, you will:

* Extract and analyze metadata
* Identify privacy risks
* Remove metadata and verify changes
* Explore browser fingerprinting
* Evaluate automated metadata sanitization tools

---
## Step 1 – Install Required Tools & Get Files

Open a terminal to Clone the lab repository:
```bash
git clone https://github.com/CYBR-3050/Lab6
```

Then install required tools:
```bash
sudo apt update 
sudo apt install -y exiftool exiv2 poppler-utils mat2
```

We need to run a tool, through the following command, to inspect each file’s internal signature (not just its filename extension). This produces a reliable description of the file format for every item in the directory so you can be sure you’re analyzing images, PDFs, or Office files and not a disguised or corrupted file:

```bash
cd Lab6
cd Content
file *
```

Similarly, computing and saving a SHA-256 fingerprint for each file will help serve as an authoritative snapshot of the files’ contents before any modification. They will let you prove later which files changed and ensure you can reconstruct an audit trail

```bash
sha256sum * > ../before_hashes.txt
```
Inventory Files:
```bash
ls -R
```
---

### **Reflection Questions**

* Why should analysts verify file type rather than relying on file extensions?
* How could disguised file types be used in an attack?
* What does a hash value represent?
* Why is hashing important in forensic investigations?

---

## Step 2 – Extract Image Metadata
Run a metadata extraction utility across the image set and capture the full output to a file. The output lists embedded fields (timestamps, device info, user tags, software, etc.) so you can examine every hidden attribute that might leak privacy-sensitive data. Use the following command to achieve this:

```bash
exiftool *.jpg *.jpeg *.png | tee ../exif_all.txt
```

Use a targeted extraction mode that isolates geolocation fields and writes them to a separate record. GPS coordinates are high-risk data. This step isolates location-related tags so they’re easy to demonstrate and compare before/after sanitization. Use the following command to achieve this:

```bash
exiftool -gps:all -a -G0:1 -s *.jpg | tee ../exif_gps.txt
```
---

### **Reflection Questions**

* What types of metadata fields are most common?
* Which fields pose the greatest privacy risk?
* Why is location data particularly sensitive?


## Step 3 – Identify Device and Owner Information
We want to dive deeper into this metadata and query the most relevant maker/owner fields to reveal device identifiers and author information. These fields can link images to specific people or devices and are often used to correlate disparate data sources:

```bash
exiftool -Make -Model -SerialNumber -OwnerName -Artist *.jpg
```
Before we start manipulating the files for further analysis, create a duplicate of the selected image so your original remains intact. Preserving originals is a forensic best practice - never overwrite the evidence you may need to reference later.

Run a bulk metadata removal operation against the copy so that embedded tags are cleared. This simulates the privacy-preserving step you would perform before public distribution of sensitive images.

***Replace <photo> with the name of an image***
```bash
cp <photo>.jpg cleaned_photo.jpg
exiftool -all= -overwrite_original cleaned_photo.jpg
```

Re-inspect the cleaned file to confirm that the previously observed metadata fields are gone, and append the file’s new SHA-256 fingerprint to the post-cleaning record. Comparing before/after fingerprints demonstrates precisely what changed and supports an audit trail:

```bash
exiftool cleaned_photo.jpg
sha256sum cleaned_photo.jpg >> ../after_hashes.txt
```

---

### **Reflection Questions**

* How could device metadata be used in an investigation?
* Why should original files always be preserved?
* What risks exist when modifying original evidence?
* How do hashes confirm file changes?

## Step 4 – Analyze PDF Metadata

Use two different tools to collect metadata from your PDFs. The first provides structural document properties (like title, author, creation date), while the second reveals deeper embedded metadata fields. Capturing the results into text files ensures you can compare them later. Use the following commands to achieve this:

```bash
pdfinfo *.pdf | tee ../pdfinfo.txt
exiftool *.pdf | tee ../pdf_exif.txt
```

Make a duplicate of the PDF to preserve the original, then run a sanitization pass that explicitly clears common identifying fields (title, author, company, keywords, timestamps).

Replace ***<doc>.pdf*** with the file name from one of the PDF files in the samples you previously downloaded. Try both files – is there any difference in the outputs?

```bash
cp <doc>.pdf redacted.pdf
exiftool -Title= -Author= -Creator= -Producer= -AllDates= -overwrite_original redacted.pdf
```
---

### **Reflection Questions**

* What types of metadata exist in PDFs?
* Why are PDFs commonly used for data leakage?
* Did any metadata remain?

---

## Step 5 – Analyze Office File Metadata
Doing the same for the Docx file, first confirm the file type (Office formats are zipped XML containers), then run metadata extraction to collect author names, company affiliations, and other hidden details. These values can unintentionally leak organizational or personal information. This can be done using the following command:

```bash
file *.docx *.pptx *.xlsx
exiftool <doc>.docx | tee ../office_exif.txt
```

Finally, as with PDFs, copy the original before sanitizing. Then clear common authoring and corporate fields. Record a new hash afterward to document the change. This demonstrates metadata removal for business files, which are often circulated publicly:
```bash
cp <doc>.docx sanitized.docx
exiftool -Author= -Company= -Manager= -AllDates= -overwrite_original sanitized.docx
sha256sum sanitized.docx >> ../after_hashes.txt
```

---

### **Reflection Questions**

* Why do Office files often contain sensitive metadata?
* How could this impact organizations?
* What organizational data might be exposed through Office metadata?
* Why is this especially dangerous in business environments?


## Step 6 – Browser Fingerprinting Analysis

Open Firefox or Chrome in your VM and visit EFF’s Cover Your Tracks (https://coveryourtracks.eff.org). Record what the site detects about your system (IP, browser version, screen size, fonts, plugins, Do Not Track status). This baseline shows how websites can profile your environment even without cookies.

Visit:
```
https://coveryourtracks.eff.org
```

Record:

* IP address
* Browser version
* Screen size
* Tracking protection status


Change your browser privacy settings to Strict Privacy/Tracking Protection. Run the fingerprint test again. Note which items changed or disappeared compared to baseline. This demonstrates the effect of built-in privacy controls. See the instructions below how to achieve this for each browser:

For Firefox:  https://support.mozilla.org/en-US/kb/enhanced-tracking-protection-firefox-desktop 
For Chrome: https://support.google.com/chrome/answer/2790761 


Open a private window and repeat the fingerprint test. Compare results to see how much private browsing mode helps, you’ll likely find it blocks cookies but doesn’t eliminate fingerprinting signals.

---

### **Reflection Questions**

* How much information can a website learn without cookies?
* Why is browser fingerprinting difficult to prevent?
* What changed after enabling stricter settings?
* Why are built-in protections not sufficient?
* What does private browsing actually protect?
* What does it fail to protect?

---


## Step 7 – Explore MAT2
Use MAT2 to perform batch scrubbing on our files we manipulated in the previous steps.

Display the types of files MAT2 can clean. This helps you understand the tool’s scope and its limitations across different file types. Use the terminal you previously used to run the following command in the ~/Lab6/Content location:
```bash
mat2 --list
```

View the metadata MAT2 would remove for a given file without altering it. This step provides insight into what the tool considers “safe to remove.”:
```bash
mat2 --show <file>
```
Duplicate the entire folder before cleaning. This preserves the originals as evidence while giving you a safe workspace to test batch cleaning.

Execute MAT2 against every file in the directory. This automates metadata scrubbing across multiple formats, showing how large sets of files can be cleaned quickly:

```bash
cd .. 
cp -r Content Content_clean
cd Content_clean
mat2 *
```

Re-run metadata extraction on the cleaned files to confirm removal. Compare the cleaned output against the originals to evaluate MAT2’s effectiveness and identify any fields that remain:
```bash
exiftool *
```
Before we finish, we will compare the hashes from the files we cleanup in this lab. First, you should create hashes of the “after” files and then compare the “before” hashes, with the “after” hashes:
```bash
sha256sum * > ../after_hashes.txt
cat ../before_hashes.txt
cat ../after_hashes.txt
```
---
### **Reflection Questions**

* What file types are supported?
* What limitations might exist?
* What metadata does MAT2 consider removable?
* Are there risks in removing too much metadata?
* What are the benefits of automation in security?
* What risks come with automated tools?
---

## **Submission Requirements**

Submit **one document** including:

* Answers to all reflection questions
* Key findings from each section
* Observations about tools and effectiveness
* Any challenges encountered

