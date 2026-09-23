# Fixing "Could not open /dev/vmmon" on VMware Workstation — Without Disabling Secure Boot

_A troubleshooting walkthrough for running a Kali Linux lab VM on a dual-boot (Ubuntu + Windows 11) host with Secure Boot kept enabled._

### The Setup

I run Kali Linux inside VMware Workstation on Ubuntu as my main lab environment for security learning — trying out tools, running labs, and generally breaking things in a safe, snapshot-able sandbox. The host machine dual-boots into Windows 11 from time to time, which meant Secure Boot needed to stay **enabled** (Windows 11 requires it).

That constraint turned a routine "VMware won't start" error into a slightly longer troubleshooting session — documented here in case it saves someone else the same detour.

### The Error

On launching VMware Workstation, I hit this:

![VMware vmmon error](https://claude.ai/chat/vmware-vmmon-error.png)

```
Could not open /dev/vmmon: No such file or directory.
Please make sure that the kernel module `vmmon` is loaded.
```

This is VMware's way of saying its kernel-level components — `vmmon` (virtual machine monitor) and `vmnet` (virtual networking) — aren't loaded into the running kernel.

### First Attempt: Rebuild the Modules

The standard fix is to rebuild the modules against the current kernel headers:

```bash
sudo vmware-modconfig --console --install-all
```

This compiled cleanly — no errors, all objects built, both `vmmon.ko` and `vmnet.ko` produced successfully. But the services still wouldn't start:

```
Starting VMware services:
   Virtual machine monitor                                            failed
   Virtual machine communication interface                             done
   VM communication interface socket family                            done
   Virtual ethernet                                                   failed
   VMware Authentication Daemon                                        done
Unable to start services
```

A clean build followed by a failed start is a specific signature — it usually points to the kernel **refusing** to load the module rather than the module being broken.

### Diagnosing: Secure Boot Lockdown

Two quick checks confirmed the cause:

```bash
$ mokutil --sb-state
SecureBoot enabled

$ sudo modprobe vmmon
modprobe: ERROR: could not insert 'vmmon': Key was rejected by service
```

`Key was rejected by service` is the exact error the kernel throws under Secure Boot's lockdown mode when it encounters an **unsigned** third-party module. VMware's freshly-compiled modules aren't signed by default, so the kernel refuses to load them — a security feature working exactly as intended, just not in my favor at that moment.

The usual advice online is "just disable Secure Boot." That wasn't an option here — Windows 11 requires it for the dual-boot side of this machine. So the fix was to **sign the modules myself** and get the kernel to trust that signature via MOK (Machine Owner Key) enrollment.

### The Fix: Self-Signing the Modules

#### 1. Generate a signing key pair

```bash
openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der -nodes -days 36500 -subj "/CN=VMware Module Sign/"
```

This produces a private key (`MOK.priv`) and a public certificate (`MOK.der`) — a 100-year validity so it doesn't unexpectedly expire mid-lab-session.

#### 2. Sign both kernel modules

```bash
sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 ./MOK.priv ./MOK.der /lib/modules/$(uname -r)/misc/vmmon.ko
sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 ./MOK.priv ./MOK.der /lib/modules/$(uname -r)/misc/vmnet.ko
```

`sign-file` stays silent on success. Verified with:

```bash
$ sudo modinfo /lib/modules/$(uname -r)/misc/vmmon.ko | grep -i sig
sig_id:         PKCS#7
signer:         VMware Module Sign
sig_key:        71:FB:E4:84:F2:72:43:B1:7B:DA:CF:40:FD:C9:9D:50:89:A9:6C:0E
sig_hashalgo:   sha256
signature:      15:EA:0D:61:D1:51:38:B7:22:3C:11:24:20:1C:CD:5D:7D:FE:16:E3:
```

#### 3. Enroll the public key into the kernel's trust store

```bash
sudo mokutil --import MOK.der
```

This prompts for a one-time password, then requires a reboot. On restart, a **blue MOK Manager screen** appears before GRUB — this is firmware-level, separate from the OS itself:

1. Select **Enroll MOK**
2. Select **Continue**
3. Select **Yes** to confirm
4. Enter the password set during `mokutil --import`
5. Reboot

After that reboot, Kali booted cleanly and VMware launched without the `/dev/vmmon` error — with Secure Boot still fully enabled.

### Automating It for Future Kernel Updates

Here's the catch with this approach: the MOK key stays enrolled permanently, but **every kernel update produces freshly-built, unsigned modules again.** Without re-signing, the same error returns after the next `apt upgrade` that pulls a new kernel.

To avoid repeating all of the above by hand, I moved the keys to a root-owned location and wrote a small script to handle rebuild + sign + load in one shot:

```bash
sudo mkdir -p /root/mok-keys
sudo mv MOK.priv MOK.der /root/mok-keys/
sudo chown root:root /root/mok-keys/MOK.priv /root/mok-keys/MOK.der
sudo chmod 600 /root/mok-keys/MOK.priv
```

**`/root/mok-keys/vmware-mok-sign.sh`:**

```bash
#!/bin/bash
# vmware-mok-sign.sh
# Rebuilds VMware kernel modules (vmmon/vmnet) and signs them with the
# enrolled MOK key, so they load cleanly under Secure Boot.
# Run this after any kernel update, before launching VMware Workstation.

set -e

MOK_PRIV="/root/mok-keys/MOK.priv"
MOK_DER="/root/mok-keys/MOK.der"
KVER="$(uname -r)"
MODDIR="/lib/modules/${KVER}/misc"

if [ "$EUID" -ne 0 ]; then
  echo "Please run as root (sudo $0)"
  exit 1
fi

if [ ! -f "$MOK_PRIV" ] || [ ! -f "$MOK_DER" ]; then
  echo "MOK key files not found at /root/mok-keys/. Aborting."
  exit 1
fi

echo "[1/3] Rebuilding VMware kernel modules for kernel ${KVER}..."
# vmware-modconfig also tries to start services immediately after building,
# which fails here because the modules aren't signed yet. That failure is
# expected at this stage, so we don't let it abort the script.
vmware-modconfig --console --install-all || true

echo "[2/3] Signing vmmon.ko and vmnet.ko..."
for MOD in vmmon vmnet; do
  MODPATH="${MODDIR}/${MOD}.ko"
  if [ -f "$MODPATH" ]; then
    /usr/src/linux-headers-${KVER}/scripts/sign-file sha256 "$MOK_PRIV" "$MOK_DER" "$MODPATH"
    echo "  Signed: $MODPATH"
  else
    echo "  WARNING: $MODPATH not found — skipping"
  fi
done

echo "[3/3] Loading modules..."
modprobe vmmon
modprobe vmnet

echo "Done. Verify with: lsmod | grep vmm"
```

```bash
sudo chmod 700 /root/mok-keys/vmware-mok-sign.sh
```

One quirk worth flagging: `vmware-modconfig --install-all` tries to **start** the VMware services right after building — which fails at that point since the modules aren't signed yet. That's expected mid-process, not a real failure, so the script uses `|| true` on that step to keep going instead of aborting.

Running the script end to end now produces:

```
[1/3] Rebuilding VMware kernel modules for kernel 6.14.0-36-generic...
...
Unable to start services
[2/3] Signing vmmon.ko and vmnet.ko...
  Signed: /lib/modules/6.14.0-36-generic/misc/vmmon.ko
  Signed: /lib/modules/6.14.0-36-generic/misc/vmnet.ko
[3/3] Loading modules...
Done. Verify with: lsmod | grep vmm
```

The MOK enrollment itself only needs to be done once — after a kernel update, it's just a matter of re-running this one script (then `sudo systemctl restart vmware`) rather than repeating the full signing/enrollment dance.

### Takeaways

* `make` succeeding but the service failing to **start** is a strong signal to check Secure Boot before anything else.
* `Key was rejected by service` on `modprobe` is Secure Boot's lockdown, confirmed instantly with `mokutil --sb-state`.
* Disabling Secure Boot isn't the only fix — self-signing kernel modules via MOK keeps Windows 11 dual-boot compatibility intact.
* Since this repeats on every kernel update, scripting the rebuild-and-sign step turns a five-step manual process into a single command.

For a home lab that needs to work smoothly across both a Linux-based Kali VM and an occasional Windows 11 boot, this setup gets the best of both worlds — no security posture traded away for convenience.
