# Network Programming Lab: HTTP Servers, SSH Tunnels, and SOCKS Proxies

In the lecture, we talked about VPNs and Yggdrasil. In this lab, we'll try something related, but simpler: we'll make SSH carry other network connections for us. We won't need root access, and we won't create any virtual network interfaces.

We'll start with a small HTTP server, see what happens when it listens on different addresses, and then reach it through SSH. Finally, we'll use SSH as a SOCKS proxy and see what a website knows about the computer connecting to it.

By the end of the lab, you should be able to explain what `0.0.0.0` and `127.0.0.1` mean for a listening server, what `ssh -L` and `ssh -D` do, and how a proxy or SSH tunnel differs from a VPN.

## Before we start

We will be using **two computers**:

* the **course server**, where you all have accounts and will run your HTTP servers;
* a **Windows lab machine or your own laptop**, where you will use a browser and make SSH tunnels.

Pay attention to *which* computer each command should run on. Sometimes you'll need two SSH sessions to the course server, and another terminal on your local machine.

Since the course server is shared by you all, you cannot all listen on the same HTTP port. We'll let the operating system choose a free port for each of you. However, you **can** all use port `1080` for SOCKS and `8080` for local forwarding. Why? You'll find out in a bit.

### Where to write your answers

As usual, SSH to the course server and clone this repository. Go to your clone. **Create a file called `answers.md` in the repository and open it in your editor.** For example, if you use `nano`:

```sh
nano answers.md
```

Write your observations and answers to the questions in **that file**, not in this `README.md`. You can organize it under four headings: `Part 1`, `Part 2`, `Part 3`, and `Part 4`. Save the file, and reopen it whenever the instructions ask you to record something. If you use `nano`, save with **Ctrl+O**, Enter, and exit with **Ctrl+X**.

Please don't leave all the writing until the end. It's easier to record what happened while you still remember it. A sentence or two for each question is enough.

## Part 1 — Start your own HTTP server

**1.** On the **course server**, in your clone of this repository, create a directory for a small webpage:

```sh
mkdir -p www
```

Using your editor, create `www/index.html`. It can be as simple as this:

```html
<!doctype html>
<html>
  <body>
    <h1>My network programming lab</h1>
    <p>Student: YOUR_USERNAME</p>
  </body>
</html>
```

Of course, replace `YOUR_USERNAME` with your own login name. You may make the page a little more interesting if you like, but please don't put passwords or private files in `www/`.

**2.** Still on the **course server**, run:

```sh
python3 -m http.server 0 --bind 0.0.0.0 --directory www
```

You may see something like:

```text
Serving HTTP on 0.0.0.0 port 41357 ...
```

The `0` in our command means *ask the operating system to choose an available port*. In this example, it chose `41357`. Your port will probably be different. **Remember your port number.** Leave this terminal running.

See? That wasn't difficult. Python comes with a small HTTP server, and it's available on the course server. Anytime you need to, you can spin up a web server in a second!

**3.** Open a **second SSH session to the course server**. Let's try connecting to your new web server. Set `PORT` to the number Python printed (not necessarily `41357`):

```sh
PORT=41357
curl -i "http://127.0.0.1:$PORT/"
```

Did you get an HTTP response and the HTML of your page? Good. The `-i` option asks `curl` to show the HTTP response headers too.

**4.** Let's try another address. Find the server's IPv4 addresses:

```sh
ip -4 addr
```

Now use the course server's address instead of `127.0.0.1`:

```sh
SERVER_IP=212.34.250.14
curl -i "http://$SERVER_IP:$PORT/"
```

This should normally work too. We told Python to bind to `0.0.0.0`, which means that it listens on all the machine's IPv4 interfaces. Notice that we are still running `curl` **on the course server**. That way, we're not yet testing whether its firewall allows incoming connections from other machines.

Do `netstat -anp | grep -i python` to see what it binds to.

**5.** Now let's try from *another computer*. On the Windows lab machine, open `cmd.exe`. If you are using your own Linux laptop, open a terminal there instead.

On Windows, run:

```cmd
curl.exe http://212.34.250.14:41357/
```

Replace `41357` with **your** port. On Linux, the same command is `curl` instead of `curl.exe`.

Does it print your HTML? If not, a firewall or another network restriction may be preventing the connection. That's okay: the two tests made directly on the course server are the important ones for this part.

Now open the same URL in Firefox (or another browser):

```text
http://212.34.250.14:41357/
```

Again, use your port. Our Python web server supports plain **HTTP**, not HTTPS. Modern browsers prefer HTTPS, so Firefox may show a warning such as **Secure Site Not Available**. If it does, look for **Continue to HTTP Site**. We're deliberately visiting our own small HTTP test server, so it's fine to continue in this exercise. If the connection itself is blocked by a firewall, clicking through an HTTPS warning won't fix that.

**Open `answers.md` in your editor now.** Under **Part 1**, write down your HTTP port and the results of all four tests: `curl` to `127.0.0.1` on the server, `curl` to the server's real address on the server, `curl` from the local machine, and Firefox from the local machine. Which worked? Which didn't? If Firefox showed a warning, what did it say?

## Part 2 — What if we listen only on localhost?

**1.** Go back to the terminal running Python and stop the HTTP server with **Ctrl+C**.

**2.** Let's start it again, but change one thing:

```sh
python3 -m http.server 41357 --bind 127.0.0.1 --directory www
```

Use **the same port you got in Part 1**, not necessarily `41357`. Notice that we've changed `0.0.0.0` to `127.0.0.1`.

**3.** In your second SSH session on the course server, repeat our two `curl` tests:

```sh
curl -i "http://127.0.0.1:$PORT/"
curl --connect-timeout 3 -i "http://$SERVER_IP:$PORT/"
```

What happened? The first connection should work, but the second should fail. Why? The HTTP server is no longer listening on the server's other IPv4 addresses. (`--connect-timeout 3` just stops `curl` from waiting too long.)

**4.** Try the server's address from the Windows lab machine or your own laptop again, using `curl` or Firefox. If it worked in Part 1, it should now fail. If a firewall already blocked it in Part 1, that's not very informative; use the two `curl` tests **on the server** to see the difference.

**5.** Back on the course server, let's ask the operating system what is listening on your port:

```sh
ss -ltn | grep ":$PORT "
```

You should see `127.0.0.1` rather than `0.0.0.0`.

**Open `answers.md` and write under Part 2:** which connections worked after the change? What listening address did `ss` show? Then answer these questions:

* What does `0.0.0.0` mean when a server binds to it?
* What changes when we bind the server to `127.0.0.1`?
* Does listening on `127.0.0.1` mean that **only your Unix user account** can connect to it? Remember that you all share the same course server. Explain.

**Don't stop your HTTP server yet.** We'll need it in the next part.

## Part 3 — Reach the server through an SSH tunnel (`-L`)

Our HTTP server is now listening only on `127.0.0.1` on the **course server**. We cannot connect directly to it from the lab machine. Can SSH help us?

**1.** On the Windows lab machine or your own laptop, first try opening:

```text
http://127.0.0.1:8080/
```

Unless you already have a local service on that port, there should be nothing there. Notice that `127.0.0.1` now means **your local machine**, not the course server.

**2.** Open `cmd.exe` on Windows (or a terminal on Linux) and run:

```sh
ssh -N -L 127.0.0.1:8080:127.0.0.1:41357 YOUR_USERNAME@212.34.250.14
```

Replace `YOUR_USERNAME` with your course server login and `41357` with **your HTTP port**.

The command may appear to do nothing. It doesn't give you a shell prompt, and that's normal. The `-N` option tells SSH not to run a remote command. Instead, it keeps the connection open for forwarding. Leave this terminal running.

**3.** Try the browser again:

```text
http://127.0.0.1:8080/
```

It's your webpage! But your Python server is listening only on the course server's loopback interface. How did the browser reach it? SSH is listening on port `8080` **on your local machine** and forwarding connections through the SSH session to your HTTP server.

If you get an HTTPS warning, continue to the HTTP page as in Part 1.

**4.** Stop the SSH tunnel with **Ctrl+C** and reload the page. What happens now? You can start the tunnel again if you want to check.

**Open `answers.md` and write under Part 3:** did the page work before, during, and after the SSH tunnel? Then answer:

* On which machine is port `8080` listening?
* On which machine is your Python HTTP port listening?
* What does the **second** `127.0.0.1` in the `ssh -L` command refer to?
* Why can the browser reach a loopback-only service through this tunnel?

Take a screenshot while the tunnel is working. Make sure the browser shows both `http://127.0.0.1:8080/` and your username on the page. Save it as **`http-tunnel.png`**. We'll put it in `evidence/` in the repository before submitting.

## Part 4 — SSH as a SOCKS proxy (`-D`)

With `ssh -L`, we forwarded one local port to one particular service. But what if we want to send connections to *different* destinations through the course server? Let's try a SOCKS proxy.

**1.** On your local machine, first make sure Firefox is **not** using a manual proxy. Open:

<https://ip.me>

What public IP address does the website show? This is the public address from which your connection appears to come. You don't need to share your full personal IP address with us; you may redact part of it in `answers.md`.

**2.** Open `cmd.exe` on Windows (or a terminal on Linux), and run:

```sh
ssh -N -D 127.0.0.1:1080 YOUR_USERNAME@212.34.250.14
```

Use your own username. As before, leave SSH running. This time, `-D` creates a SOCKS proxy on **your local machine**.

**3.** In Firefox, go to **Settings → Network Settings → Settings**, and select **Manual proxy configuration**. Set:

| Setting | Value |
| --- | --- |
| SOCKS Host | `127.0.0.1` |
| Port | `1080` |
| SOCKS version | `SOCKS v5` |
| Proxy DNS when using SOCKS v5 | Enabled |

You don't need to fill in the HTTP or HTTPS proxy fields.

**4.** Now reload <https://ip.me>. What address do you see? Does it look familiar? It's normally the public address used by the course server for outgoing connections. It may or may not be exactly the same as the address on the server's network interface.

**5.** Turn off the manual SOCKS proxy in Firefox and reload the page. Did the old public IP address return? You can enable the proxy again to check.

**6.** We can also test this without a browser. While `ssh -D` is running, open **another local terminal**. On Windows, try:

```cmd
curl.exe https://api.ipify.org
curl.exe --socks5-hostname 127.0.0.1:1080 https://api.ipify.org
```

On Linux, use `curl` instead of `curl.exe`.

The first command connects normally. The second asks `curl` to use the SOCKS proxy, including for hostname resolution. Compare the two addresses. If both happen to be the same, it might be because the lab machine and course server share a public exit address; that alone doesn't prove the proxy is broken.

**Open `answers.md` and write under Part 4:** the public IP addresses you saw with and without SOCKS, and what happened when you turned the proxy off again. If you ran the optional `curl` test, write down its results too. Then answer:

* Is port `1080` listening on your local machine or the course server?
* Does configuring a SOCKS proxy in Firefox change the traffic of **every** application on your computer?
* Is the connection between your computer and the SSH server encrypted? What about the connection from the SSH server to the destination website?
* How is this different from a VPN such as WireGuard? And how is it different from a network such as Yggdrasil?

Take one more screenshot, this time of <https://ip.me> **with the SOCKS proxy enabled**. Save it as **`socks-ip.png`**. You don't need a screenshot of your original, unproxied address.

## Copy your screenshots to the server using SCP

You have your screenshots on your Windows lab machine (or your own laptop), but your Git repository is on our server.

How do you get the files there?

Well, SSH can do more than give you a remote shell or create tunnels. It can also transfer files.

There is a command called `scp`, which stands for *secure copy*. It uses SSH to copy files between computers.

Let's try it.

**1.** First, on the course server, go to your lab repository and create a directory for screenshots:

```sh
mkdir -p evidence
```

**2.** On your Windows machine, save your screenshots to your Desktop, naming them `http-tunnel.png` and `socks-ip.png`.

Open `cmd.exe` and check whether you have `scp`:

```cmd
where scp
```

You should find `scp.exe`. Windows includes it with the OpenSSH client.

**3.** Now let's copy the first screenshot to the server. Run this command **on your Windows machine, not on the server**:

```cmd
scp.exe "%USERPROFILE%\Desktop\http-tunnel.png" YOUR_USERNAME@the.hell.am:YOUR_REPO/evidence/
```

Replace `YOUR_USERNAME` with your username on the server and `YOUR_REPO` with the directory name of your cloned lab repository in your home directory.

Enter your SSH password if asked.

Now copy the second screenshot:

```cmd
scp.exe "%USERPROFILE%\Desktop\socks-ip.png" YOUR_USERNAME@the.hell.am:YOUR_REPO/evidence/
```

Notice the colon (`:`) after the hostname. It separates the remote computer's name from the destination path.

**4.** Go back to your SSH session on the server. In your lab repository, type:

```sh
ls -l evidence/
```

Are your screenshots there?

See? You can copy files between computers using SSH, without a graphical file manager.

And `scp` works in both directions. You can also copy files *from* the server *to* your computer. Can you figure out how?

**5.** Now add the screenshots to Git:

```sh
git add evidence/
git commit -m "Add lab screenshots"
git push
```

That's it. Your screenshots are now part of your lab submission.

## What to submit

You should have your webpage, your `answers.md` file, and two screenshots. **Only two screenshots are required.** Put them in your repository like this:

```text
README.md
www/
    index.html
answers.md
evidence/
    http-tunnel.png
    socks-ip.png
```

Your `answers.md` should contain the observations and answers requested **at the end of each part**. Short answers are fine, but please explain what you observed, not just "yes" or "no".

The screenshots were taken on your **local machine**, while the repository is on the **course server**. To put them in the repository, first create the evidence directory on the server, inside your clone:

```sh
mkdir -p evidence
```

Then copy the screenshots from the local machine using `scp`. For example, if your screenshots are saved in the current directory in Windows `cmd.exe`, you can run:

```cmd
scp http-tunnel.png YOUR_USERNAME@212.34.250.14:YOUR_REPO_DIRECTORY/evidence/
scp socks-ip.png YOUR_USERNAME@212.34.250.14:YOUR_REPO_DIRECTORY/evidence/
```

Replace `YOUR_REPO_DIRECTORY` with the name of your repository directory **inside your home directory on the course server**. If your images are somewhere else on Windows, first `cd` to the folder where you saved them, or give `scp` their full paths. The same `scp` syntax works in a Linux terminal.

Check on the course server that both images are in `evidence/`. Then **commit and push** your webpage, `answers.md`, and the screenshots to your repository. Do not upload passwords, SSH private keys, or other personal files.

## Before you leave

* Stop your Python HTTP server with **Ctrl+C**.
* Stop any SSH tunnels you still have running, also with **Ctrl+C**.
* Restore Firefox's original proxy settings. Otherwise you may wonder later why the browser cannot connect to anything!

## A few hints

* If Python says `Address already in use`, perhaps somebody started using your old port. Start it with port `0` again, take note of the new port, and use it in the remaining commands.
* If local ports `8080` or `1080` are already used on your machine, choose other local ports and update your commands (and Firefox settings) accordingly.
* If `http://127.0.0.1:8080/` doesn't work through SSH, first check that Python is still running on the course server and that you've used **your** assigned HTTP port in `ssh -L`.
* `127.0.0.1` refers to the machine **on which it is used**. The course server has its own loopback interface, and your laptop has another one.
* Binding a service to `127.0.0.1` does **not** make it private to your user account. Other users logged in to the same server may still connect to it.
* Python's little HTTP server is handy for experiments, but don't use it for sensitive files or production websites.
