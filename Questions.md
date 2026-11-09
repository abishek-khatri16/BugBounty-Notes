<h1 align="center">🧭 Recon & Testing Methodology</h1>
<p align="center"><b>Questions to ask before and during testing, plus a reusable report template</b></p>

<p align="center">
  <img src="https://img.shields.io/badge/Type-Methodology-blue?style=for-the-badge" alt="Type">
  <img src="https://img.shields.io/badge/Use-Reference-orange?style=for-the-badge" alt="Use">
</p>

---

## 🧩 Questions About the Application

| # | Question |
|:-:|----------|
| 1 | What stack/languages are used? |
| 2 | What server is running the application? |
| 3 | Is there a WAF? |
| 4 | What additional libraries are used? Any known exploits for them? Custom JS libraries? |
| 5 | Is there authentication? (e.g. OAuth through Google/Facebook) |
| 6 | What objects are used? |
| 7 | How is the session established? |
| 8 | Are there useful comments in the source? |
| 9 | How does it handle special characters? |
| 10 | Can you trigger any error messages? |
| 11 | What common features are present? |
| 12 | How is a user identified? |
| 13 | Are there multiple user roles? |
| 14 | Is there an API? |
| 15 | Is there a Content Management System? |
| 16 | Is there a Content Security Policy? |
| 17 | Is CORS implemented? |
| 18 | Is CAPTCHA used? |
| 19 | Are WebSockets used? |
| 20 | Is the source code publicly available? |

---

## 🖥️ Questions About the Server

| # | Question |
|:-:|----------|
| 1 | What ports are open? |
| 2 | What services are running on those ports? |
| 3 | Is it hosted in the cloud? |
| 4 | Is it hosting multiple apps using virtual hosting? |
| 5 | What is the OS? |
| 6 | Can you get the kernel version? |

---

## 📄 Questions for Every Page

| # | Question |
|:-:|----------|
| 1 | Which part of CRUD does it cover? |
| 2 | What HTTP methods can be used? (GET / POST / PUT / DELETE / etc.) |
| 3 | What parameters can be used? |

---

## 🔗 Finding Chaining Bugs

<details open>
<summary><b>1️⃣ Open Redirect?</b></summary>

If yes, check:
- Can you redirect to different paths?
- Can you redirect to
