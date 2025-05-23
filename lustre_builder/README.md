# Ansible Role: Lustre Builder

## Purpose

This Ansible role automates the process of compiling the Lustre file system from source code, generating RPM packages, and optionally signing these RPMs. It is designed to work on RHEL/CentOS and SLES-based systems.

## Requirements

-   Ansible 2.9 or later.
-   Target machine must be RHEL/CentOS or SLES based.
-   Internet access on the target machine for cloning the Lustre repository and downloading dependencies.
-   Sufficient disk space (e.g., 30GB+ recommended for source and build artifacts) and system resources (CPU, RAM) for compilation.
-   If running the role as a non-privileged user (i.e., without global `become: true` in your playbook), the paths specified by `lustre_src_path` and `lustre_rpm_path` must be writable by this user. You can override these variables to point to user-writable locations.
-   If RPM signing (`lustre_sign_rpms: true`) is enabled:
    -   The GPG private key (specified by `lustre_rpm_gpg_key`) must be available in the GPG keyring of the user executing the signing on the target machine.
    -   The `rpm-sign` package must be installable.
    -   The `community.general.rpm_sign` Ansible module must be available in your Ansible environment (`ansible-galaxy collection install community.general`).

## Role Variables

The primary variables for this role are defined in `defaults/main.yml` and can be overridden in your playbook:

-   `lustre_version`: (Default: `"master"`)
    The git branch, tag, or commit ID of the Lustre code to clone and build.
    Examples: `"master"`, `"b2_15"`, `"2.15.4"`.

-   `lustre_src_path`: (Default: `"/usr/local/src/lustre"`)
    The base directory on the target machine where the Lustre source code will be cloned. The actual checkout will be in a subdirectory named `lustre-release-{{ lustre_version }}`.

-   `lustre_rpm_path`: (Default: `"/opt/lustre_rpms"`)
    The base directory on the target machine where the compiled RPMs will be stored. RPMs for a specific version will be placed in a subdirectory named `{{ lustre_version }}`.

-   `lustre_rpm_gpg_key`: (Default: `""`)
    The GPG key ID or name to use for signing the RPMs. If `lustre_sign_rpms` is `true`, this variable must be set. Example: `"My Signing Key <myemail@example.com>"`.

-   `lustre_sign_rpms`: (Default: `false`)
    A boolean value indicating whether to sign the compiled RPMs.

Internal variables used by the role (defined in `vars/main.yml`) include:
-   `lustre_checkout_path`: Calculated full path to the Lustre source code checkout.
-   `lustre_version_rpm_path`: Calculated full path where RPMs for the specific `lustre_version` will be stored.

## Dependencies

This role will attempt to install necessary build dependencies using the system's package manager (`yum` for RedHat family, `zypper` for Suse family). This includes:
- Standard development tools (gcc, make, automake, etc.).
- `kernel-devel` package. By default, this will be for the currently running kernel. For building against a specific kernel version, ensure the corresponding `kernel-devel` is installed and that `./configure` picks it up (may require customizing configure flags not yet exposed as a variable).
- `rpm-sign` if RPM signing is enabled.
These package installation steps require privilege escalation (e.g., `become: true`).

## Example Playbook

Here is an example of how to use this role in a playbook (`playbook.yml`):

```yaml
---
- name: Build Lustre RPMS
  hosts: localhost # Or your target build machine, e.g., 'buildservers'
  connection: local # If hosts is localhost. Remove if targeting remote build machine.
  gather_facts: true # Good to have facts, especially for ansible_os_family

  # vars:
    # Override any defaults from the role here if needed for testing
    # lustre_version: "2.15.3" # Example: Test a specific version
    # lustre_sign_rpms: true
    # lustre_rpm_gpg_key: "Your GPG Key ID For Testing"
    # lustre_src_path: "/home/myuser/lustre_build/src" # Example for non-privileged path
    # lustre_rpm_path: "/home/myuser/lustre_build/rpms" # Example for non-privileged path

  roles:
    - lustre_builder
```

## Usage Notes

-   **Permissions and `become`**:
    -   Tasks related to package installation (`epel-release`, build dependencies, `rpm-sign`) use `become: true` and require root privileges.
    -   Other tasks (directory creation for source/RPMs, git cloning, compilation, RPM signing) are now designed to run with the privileges of the Ansible user. If you are not running the playbook with global `become: true`, ensure that the directories specified by `lustre_src_path` (default: `/usr/local/src/lustre`) and `lustre_rpm_path` (default: `/opt/lustre_rpms`) are writable by the Ansible user, or override these variables to point to user-writable paths (see example playbook).
    -   The Lustre build process itself is assumed not to write to protected system locations outside of its checkout and RPM output paths.
    -   For RPM signing as a non-privileged user, ensure the GPG key is accessible to that user.
-   **Kernel Version**: Lustre compilation is sensitive to the `kernel-devel` package version. The role installs `kernel-devel` for the running kernel by default. If you need to build against a specific, different kernel, you may need to ensure that version of `kernel-devel` is installed prior to running this role or customize the configure step.
-   **RPM Signing**:
    -   To sign RPMs, the private GPG key must be imported into the GPG keyring of the user performing the signing on the build machine. If not using `become: true` for the signing task, this is the Ansible user.
    -   The public part of this GPG key should be distributed and imported into the RPM database of any system where these RPMs will be installed and verified (`rpm --import /path/to/your_public_key.asc`).
    -   If your GPG key is protected by a passphrase, you may need to use `gpg-agent` or configure the `passphrase_file` option in the `rpm_sign` task (not currently exposed as a variable).
-   **Compilation Time**: Lustre compilation can be a lengthy process. Be prepared for the Ansible playbook to run for a significant amount of time, especially on less powerful machines or for full Lustre releases.
-   **Idempotency**: Most steps are idempotent. However, `make rpms` might recompile if run again. If RPMs for a version already exist in `lustre_version_rpm_path`, they might be re-signed if signing is enabled.
