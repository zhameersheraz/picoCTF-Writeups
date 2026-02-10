Inspect HTML – picoCTF Writeup
Challenge: Inspect HTML
Category: Web Exploitation
Difficulty: Easy
Points: Not specified
Flag: picoCTF{1n5p3t0r_0f_h7ml_fd5d57bd}
________________________________________
Description
This challenge provides a simple webpage and asks whether you can find the hidden flag. The task is to inspect the website and discover anything unusual within the page source.
________________________________________
Approach
Since the challenge title is “Inspect HTML,” it strongly suggests that the flag is hidden directly in the webpage’s source code rather than behind complex functionality.
A common beginner web exploitation technique is checking HTML comments because developers sometimes leave sensitive information there.
________________________________________
Solution
Step 1: Open the Website
Navigate to the provided challenge URL:
http://saturn.picoctf.net:64474/
The page displayed a short passage about Histiaeus along with a source citation. Nothing on the visible page indicated the presence of a flag.
________________________________________
Step 2: Inspect the Page Source
To view the HTML source:
•	Right-click anywhere on the webpage
•	Select Inspect
OR
•	Press F12 to open Developer Tools
Inside the HTML structure, a hidden comment was discovered near the bottom:
<!-- picoCTF{1n5p3t0r_0f_h7ml_fd5d57bd} -->
________________________________________
Step 3: Capture the Flag
The flag was directly embedded inside the HTML comment. Copy it exactly as shown and submit it.
________________________________________
Why This Works
•	HTML comments are invisible to users but remain accessible in the source code.
•	Developers sometimes forget to remove test data or notes before deployment.
•	Many beginner CTF challenges rely on basic inspection skills to build foundational web security knowledge.
________________________________________
Flag
picoCTF{1n5p3t0r_0f_h7ml_fd5d57bd}
________________________________________
Tools Used
•	Browser Developer Tools (F12) – Inspect webpage source
•	Web Browser – Access the challenge instance
