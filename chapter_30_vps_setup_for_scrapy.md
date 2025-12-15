# Chapter 30: VPS Setup for Scrapy

Think about the first time you use SSH to connect to a remote server. You're staring at a black terminal screen, and you might feel terrified. All your life, you've used computers with icons, folders, and mouse pointers. Now, you're looking at a blinking white rectangle that expects you to speak its language.

You might feel like an explorer who has traveled to a distant land where nobody speaks his tongue. 

You type `ls` and see nothing. You type `dir` and get an error. You might spend the next four hours just trying to find where the "Desktop" is, only to realize that a professional server doesn't *have* a desktop. It's just a brain in a box, thousands of miles away. You might almost give up and think, "Maybe I'll just leave my laptop running in the closet. That's easier."

But then, you successfully run your first command on the server. You see a file being created in the cloud. You'll realize that this "black screen" is actually the most powerful tool you've ever owned. It's a computer that never sleeps, never runs out of battery, and has a high-speed fiber-optic connection to the rest of the world. Once you learn how to set it up properly, your scrapers won't be limited by your own house anymore; they'll be citizens of the global internet.

In this chapter, you're going to learn how to claim your piece of that digital real estate.

---

## Introduction

To run production-grade scrapers, you need a computer that is "always on." Your laptop isn't built for this. It goes to sleep, it loses Wi-Fi, and it needs to be carried in a bag. A **Virtual Private Server (VPS)** is a professional remote computer that you rent specifically to run your code.

In this chapter, we're going to build your server from scratch. We'll learn how to choose a provider, how to connect securely using SSH, and how to set up a professional Linux environment (Ubuntu) so it's ready to host your Scrapy spiders 24/7.

## What is a VPS?

Think of a VPS like a single unit in a massive apartment building. You don't own the whole building (the physical server), but you have your own private space with your own door and your own locks. Within that space, you can do whatever you want install any software, run any script, and use your own dedicated slice of the building's resources (CPU and RAM).

## Choosing a Provider

In 2026, you have four main professional choices:
1.  **DigitalOcean:** The friendliest for beginners. Their servers (called "Droplets") start at $4/month.
2.  **Hetzner:** The king of value. Based in Germany, they offer incredible power for very low prices.
3.  **AWS (Amazon Web Services):** The industry standard. Incredible range of tools, but can be complex and expensive if you aren't careful.
4.  **Vultr:** A great all-rounder with servers in dozens of countries.

**My Advice:** Start with DigitalOcean or Hetzner. They are simple to set up and very predictable.

## Initial Server Setup: The Basics

Once you rent your server, you'll get an **IP Address** and a **Password**.

### 1. Connecting via SSH
Open your terminal on your laptop and type:
```bash
ssh root@your_server_ip
```
Type `yes` when it asks about the key, then enter your password. You are now "inside" the remote machine.

### 3. Hardening SSH (Critical!)
Leaving password login enabled is a security risk. Hackers run scripts that guess thousands of passwords a minute.

**Step A:** Generate an SSH Key on your laptop (if you haven't already):
`ssh-keygen -t ed25519`

**Step B:** Copy it to the server:
`ssh-copy-id root@your_server_ip`

**Step C:** Disable Password Login on the server:
Edit `/etc/ssh/sshd_config` and set:
```
PasswordAuthentication no
PermitRootLogin prohibit-password
```
Then restart SSH: `service ssh restart`. Now, only YOU can log in.

### 2. Basic Security
A server on the public internet will be attacked within minutes. You MUST do these two things:
*   **Update the system:** `apt update && apt upgrade -y`
*   **Install a firewall:** `ufw allow ssh && ufw enable`

### 4. Fail2Ban
For extra protection, install `fail2ban`. It watches for repeated failed login attempts and bans the attacker's IP automatically.
```bash
sudo apt install fail2ban -y
```

## Installing the Stack

Your server comes with almost nothing installed. You need to build your environment just like you did on your laptop.

```bash
# Install Python and Git
sudo apt install python3-pip python3-venv git -y

# Create your project space
mkdir projects
cd projects
git clone https://github.com/your-repo/my-scraper.git

# Set up the environment
python3 -m venv venv
source venv/bin/activate
pip install scrapy psycopg2-binary # and any others you need
```

> [!WARNING]
> **Scrapy Doc Gap: The UTF-8 Terminal Trap**
> Some VPS providers give you a "minimal" Linux install where the system language isn't set to UTF-8. If your scraper hits a page with emojis or special characters, Scrapy might crash with a `UnicodeEncodeError` when trying to print to the log. 
> 
> To fix this, add `export LC_ALL=C.UTF-8` to your `.bashrc` or your shell scripts. This tells the server to handle all languages properly and prevents your scraper from dying over a simple "heart" emoji.

## Automation with Ansible (Infrastructure as Code)

Setting up servers manually involves a lot of typing. If you need 10 servers, it's a nightmare.
**Ansible** allows you to write a "Playbook" (a YAML file) that describes your perfect server.

```yaml
# playbook.yml
- hosts: spiders
  tasks:
    - name: Install Scrapy
      pip:
        name: scrapy
        state: latest
```
You run one command: `ansible-playbook playbook.yml`, and Ansible connects to 10 servers and sets them all up simultaneously. It is the professional standard for deployment.

## Managing Processes with Systemd

If you run `scrapy crawl quotes` on your server and then close your terminal, the spider will stop. You need a way to tell the server: "Keep this running in the background even if I'm not here."

The professional way to do this is using **Systemd**. You create a simple text file (a "Unit file") that describes your spider:

```text
# /etc/systemd/system/myspider.service
[Unit]
Description=My Daily Scraper

[Service]
WorkingDirectory=/root/projects/my-scraper
ExecStart=/root/projects/my-scraper/venv/bin/scrapy crawl quotes
Restart=always

[Install]
WantedBy=multi-user.target
```

Now you can start your spider with: `systemctl start myspider`. The server will now handle the rest!

## Chapter Summary

**What we covered:**
- A VPS is a professional, "always-on" remote server for your scrapers.
- Choose a provider like DigitalOcean or Hetzner for a balance of price and simplicity.
- Connect securely using SSH and always perform initial system updates.
- Set up a clean environment with a `venv`, just like on your local machine.
- Use **Systemd** to manage your scrapers as background services that survive terminal closures and server reboots.

**Key code:**
```bash
ssh user@ip_address
sudo systemctl start my_service
```

**Previous chapter:**
[Chapter 29: Hardening for Production](./chapter_29_hardening_for_production.md)

**Next chapter:**
Your server is up. Your spider is running. But how do you know if it's actually working? What if it hits a wall while you're asleep? In the next chapter, we're going to explore **Monitoring and Logging** building a professional dashboard to keep an eye on your spiders 24/7.

---

**End of Chapter 30**
