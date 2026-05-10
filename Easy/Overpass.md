Preambule
Overpass
Easy room
Linux/Web

1) Reconassanse
   1.1) Open ports
     nmap -sS <IP-HOST>
     <img width="523" height="185" alt="image" src="https://github.com/user-attachments/assets/53d9cc16-42b9-4a30-9f9a-e96213042925" />
     Getting 22 and 80 ports
   1.2) Reveiling hidden directories
      gobuster dir -u <IP-HOST> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x html,js,php 
     <img width="902" height="552" alt="image" src="https://github.com/user-attachments/assets/e68107a7-d76f-42b5-a656-a7deba45c9f5" />
     Getting admin.html with code 200 and admin with code 301(restricted)
   1.3) Navigate to admin.html and analyze source code
     <img width="622" height="232" alt="image" src="https://github.com/user-attachments/assets/3e3f9233-6b53-4037-83a2-0a84e9b35d4f" />
      Find /login.js link
   1.4) Analyze login.js logic
   <img width="432" height="128" alt="image" src="https://github.com/user-attachments/assets/00e43460-1307-4c3d-b9c2-76821cddddff" />
    Found that, it compares Cookies to string "Incorect Credentials"
   1.5) We can bypass this comparing by setting Cookies to any another string
   Open console and write
   Cookies.set("SessionToken", "")
  Now we can enter /admin
  1.6) Here we found private rsa key with username james
