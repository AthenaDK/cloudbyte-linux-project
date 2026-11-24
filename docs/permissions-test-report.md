# CloudByte Permissions Test Report: Section 2
Date: 2026-11-24
Author: Christian Hansen

## Overview & Changes

## Background
Previously, when CloudByte users created files within shared team directories, the files defaulted to the creator's personal primary group. As a result, only the individual creator had full access, breaking multi-user file sharing and cross-team collaboration across the organization.

## Solution
Today's configuration updates resolve this issue retroactively for existing files and dynamically for all newly created files:

1. **Shared Team Directories (`setgid` Mode `2770`):** Shared directories now enforce group ownership via `setgid`. Files created inside `/shared/<team>` automatically inherit the directory's team group rather than the creator's primary group.
2. **Secure Dropzone (`1773` + Sticky Bit):** The `/shared/dropbox` directory allows all employees to submit files, but directory listing (`ls`) is restricted to administrators (admins). Non-administrative users cannot browse other users' submissions, and the sticky bit prevents users from deleting files owned by others.

## Verification Test Matrix

| User  | Command / Action                                    | Expected | Observed | Verification Details |
|-------|-----------------------------------------------------|----------|----------|----------------------|
| alice | `touch /shared/engineering/alice-file.txt`         | Allow    | Allow    | File tagged with group `engineering` (`setgid`) |
| bob   | `touch /shared/engineering/bob-file.txt`           | Allow    | Allow    | File tagged with group `engineering` (`setgid`) |
| admin | `find /shared/engineering -type f -group engineering`| All files| All files| `setgid` successfully propagated to all files |
| emma  | `ls /shared/dropbox`                                | Deny     | Deny     | Directory listing blocked for non-admins |
| emma  | `echo "data" > /shared/dropbox/emma-note.txt`       | Allow    | Allow    | Dropzone write access functions correctly |
| emma  | `cat /shared/dropbox/bob-note.txt`                  | Allow    | Allow    | Path known; POSIX blocks listing, not direct file read |
| emma  | `rm /shared/dropbox/bob-note.txt`                   | Deny     | Deny     | Sticky bit (`+t`) prevents cross-user deletion |
| kate  | `ls /shared/dropbox`                                | Allow    | Allow    | Administrator full directory visibility |
| kate  | `cat /shared/dropbox/emma-note.txt`                 | Allow    | Allow    | Administrator full read access |

## Engineering Notes & POSIX Caveats

- **Setgid Inheritance (Mode `2770`):** Enforcing `setgid` makes "which files does the engineering team own?" a clean, answerable question for audit scripts. Without `setgid`, file ownership defaults to the creator's personal primary group, breaking team-level permissions.
- **Sticky Bit Protection (Mode `1773`):** The sticky bit (`+t`) ensures that even in a write-heavy directory, users can only delete or rename files they personally own.
- **POSIX Permission Boundary Caveat:** Standard Linux mode `1773` (`rwxrwx-wx`) prevents non-admin users from listing (`ls`) directory contents, but does not prevent reading (`cat`) a file if the exact file path is already known. Completely isolating file reads in a shared dropzone requires advanced ACLs or dedicated security frameworks beyond standard POSIX permissions.