# nmfuzz
CTF Enumeration Automation Tool. A bash script that automates the entire nmap scan, tcp (SYN scan) and udp. Detects http/https servicies and launches both directory (gobuster) and subdomain (wfuzz) fuzzing.

---------
# Notes

The script isn't finished.
  - The webscan command is broken if the machine is not able to resolve the domain name. Needs to update /etc/hosts.
  - Subdomain scan not tested yet.
  - More preconfigurable variables need to be implemented.

README will be done once the script is fully functional.
