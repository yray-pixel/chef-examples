

## Adversarial Review Findings

**Agent:** Privilege Escalation Gate

**Summary:** The nginx Ansible role performs system-level operations without declaring privilege escalation. Package installation, service management, and writing to /etc/nginx all require root. Tasks missing 'become: yes' will fail on hardened hosts where the connection user is unprivileged.

### [CRITICAL] ansible/roles/nginx/tasks/main.yml

Package installation tasks require root but 'become: yes' is not declared at task or role level

**Evidence:**
```
- name: Install the nginx packages
  ansible.builtin.dnf:
    name: "{{ item }}"
    state: present
  loop: "{{ redhat_pkg }}"
  when: ansible_facts['os_family'] == "RedHat"
```

### [CRITICAL] ansible/roles/nginx/tasks/main.yml

Writing to /etc/nginx/nginx.conf requires root; no privilege escalation declared

**Evidence:**
```
- name: Copy the nginx configuration file
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: "0644"
  notify:
    - Restart nginx
```

### [WARNING] ansible/roles/nginx/handlers/main.yml

Service restart handler requires root without explicit privilege escalation

**Evidence:**
```
- name: Restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

### [WARNING] ansible/roles/nginx/tasks/main.yml

EPEL repo file installation requires root; task relies on become inherited from play

**Evidence:**
```
- name: Copy epel repo file
  ansible.builtin.copy:
    src: epel.repo
    dest: /etc/yum.repos.d/epel.repo
    mode: "0644"
```

---

## Adversarial Review Findings

**Agent:** Idempotency Auditor

**Summary:** Several tasks in the nginx role are not fully idempotent. Shell commands used for configuration validation run unconditionally, and the EPEL repo file is copied without a checksum check, causing unnecessary changes on repeated runs.

### [WARNING] ansible/roles/nginx/tasks/main.yml

nginx -t config validation runs as a shell command on every play execution

**Evidence:**
```
- name: Validate nginx configuration
  ansible.builtin.shell: nginx -t
  changed_when: false
```

### [WARNING] ansible/roles/nginx/tasks/main.yml

EPEL repo file copied unconditionally — triggers 'changed' even when content is identical

**Evidence:**
```
The ansible.builtin.copy module is idempotent by checksum, but the source file
epel.repo is not pinned to a specific version or checksum. If the file on the
controller changes between runs (e.g. repo URL update), the task reports
'changed' without any explicit intent by the operator.
```

### [WARNING] ansible/roles/nginx/tasks/main.yml

Service start task not guarded — reports 'changed' when nginx is already running

**Evidence:**
```
- name: Start the nginx service
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

### [WARNING] ansible/roles/nginx/defaults/main.yml

Default variable nginx_worker_processes set to 'auto' but overridden unconditionally

**Evidence:**
```
When nginx_worker_processes is set in host_vars, the template renders a new
nginx.conf on every run because the rendered value is compared byte-for-byte
against the file on disk. A notify on the template task causes an nginx restart
even when the effective configuration is unchanged.
```

---
