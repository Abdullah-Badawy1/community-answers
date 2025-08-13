# Fix: Hyprlock Password Authentication Issues

A comprehensive solution for hyprlock not accepting the correct password after failed authentication attempts.

## 🚨 The Problem

Many users experience hyprlock becoming unresponsive to the correct password after typing it wrong a few times. This happens because hyprlock uses PAM (Pluggable Authentication Modules) which includes `pam_faillock.so` - a security module that temporarily locks accounts after multiple failed authentication attempts.

## 🔍 Root Cause Analysis

By default, hyprlock's PAM configuration (`/etc/pam.d/hyprlock`) includes the system login configuration, which eventually references `pam_faillock.so`. This module locks your account for a period after 3+ failed attempts, even for screen unlock attempts.

**Configuration Chain:**
```
/etc/pam.d/hyprlock 
    → includes login
    → includes system-local-login
    → includes system-login  
    → includes system-auth
    → contains pam_faillock.so ← THE CULPRIT
```

## ✅ The Solution

Create a custom PAM configuration for hyprlock that removes the faillock mechanism while maintaining security.

### Step 1: Backup the Original Configuration

```bash
sudo cp /etc/pam.d/hyprlock /etc/pam.d/hyprlock.backup
```

### Step 2: Create the New Configuration

```bash
sudo tee /etc/pam.d/hyprlock << 'EOF'
# PAM configuration file for hyprlock
# Custom configuration without faillock to always accept correct password

auth       required   pam_shells.so
auth       requisite  pam_nologin.so
-auth      [success=2 default=ignore]  pam_systemd_home.so
auth       [success=1 default=bad]     pam_unix.so          try_first_pass nullok
auth       optional   pam_permit.so
auth       required   pam_env.so

account    required   pam_access.so
account    required   pam_nologin.so
-account   [success=1 default=ignore]  pam_systemd_home.so
account    required   pam_unix.so
account    optional   pam_permit.so
account    required   pam_time.so

password   include    system-auth

session    optional   pam_loginuid.so
session    optional   pam_keyinit.so       force revoke
-session   optional   pam_systemd_home.so
session    required   pam_limits.so
session    required   pam_unix.so
session    optional   pam_permit.so
session    optional   pam_lastlog2.so      silent
session    optional   pam_motd.so
session    optional   pam_mail.so          dir=/var/spool/mail standard quiet
session    optional   pam_umask.so
-session   optional   pam_systemd.so
session    required   pam_env.so
EOF
```

### Step 3: Clear Any Existing Locks (Optional)

```bash
sudo faillock --user $(whoami) --reset
```

## 🎯 What This Configuration Does

| Component | Action | Result |
|-----------|--------|---------|
| **Removes** | All `pam_faillock.so` modules | No more account lockouts |
| **Keeps** | Essential authentication via `pam_unix.so` | Still secure password verification |
| **Maintains** | All other PAM functionality | Sessions, environment variables, etc. work normally |
| **Result** | hyprlock always accepts correct password | No matter how many failed attempts |

## 🔒 Security Considerations

| ✅ Secure | ⚠️ Consider |
|-----------|-------------|
| Password authentication still secure through `pam_unix.so` | Change only affects hyprlock, not system login or sudo |
| All other PAM security features remain active | Removes temporary lockout mechanism for screen unlock |
| No reduction in actual password security | Consider if this fits your security model |

## 🧪 Testing the Fix

After making these changes:

1. **Lock your screen** with hyprlock
2. **Type wrong passwords** several times (3+ times)
3. **Type the correct password** - it should work immediately ✅

### Expected Behavior:
- ❌ **Before**: Correct password rejected after 3+ wrong attempts
- ✅ **After**: Correct password always accepted

## 🔄 Reverting Changes

If you need to restore the original behavior:

```bash
sudo cp /etc/pam.d/hyprlock.backup /etc/pam.d/hyprlock
```

## 🚑 Emergency Quick Fix

If you're currently locked out and need immediate access:

```bash
sudo faillock --user $(whoami) --reset
```

This temporarily clears the lock without changing the configuration.

## 📋 System Requirements

- **OS**: Linux with systemd (tested on Arch Linux)
- **Access**: sudo/root privileges required
- **Dependencies**: hyprlock using PAM authentication
- **PAM Version**: Compatible with modern PAM implementations

## 🤔 Why This Happens

This behavior is actually **expected PAM security functionality**:

- **Purpose**: Protects against brute force attacks on system login
- **Implementation**: `pam_faillock.so` temporarily locks accounts after failed attempts
- **Problem**: Screen unlock context doesn't need this protection (you're already at your machine)
- **Solution**: Remove faillock for hyprlock while keeping it for system security

## 🐛 Troubleshooting

### Issue: Permission Denied
```bash
# Solution: Use sudo
sudo tee /etc/pam.d/hyprlock << 'EOF'
[configuration content]
EOF
```

### Issue: Configuration Not Taking Effect
```bash
# Check if file was created correctly
cat /etc/pam.d/hyprlock

# Restart any related services if needed
sudo systemctl restart systemd-logind
```

### Issue: Still Getting Locked Out
```bash
# Clear existing locks
sudo faillock --user $(whoami) --reset

# Verify configuration is correct
diff /etc/pam.d/hyprlock.backup /etc/pam.d/hyprlock
```

## 📚 Understanding the Configuration

### Key PAM Modules Explained:

- **`pam_unix.so`**: Handles actual password verification (kept)
- **`pam_faillock.so`**: Manages account lockouts (removed)
- **`pam_shells.so`**: Checks valid user shells (kept)
- **`pam_nologin.so`**: Checks system-wide login restrictions (kept)

### Configuration Syntax:
```
control_flag    module_name    [module_arguments]
```

- **`required`**: Must succeed for authentication to continue
- **`optional`**: Success/failure doesn't affect final result
- **`-`prefix**: Module failure won't break authentication stack

## 🤝 Contributing

Found an issue or improvement? Feel free to:
- Open an issue
- Submit a pull request
- Share your experience

## 📄 License

This configuration and documentation are provided as-is for educational and practical purposes. Use at your own discretion and ensure it fits your security requirements.

---

**Last Updated**: 2025-08-13  
**Tested On**: Arch Linux with hyprlock  
**Status**: ✅ Working Solution
