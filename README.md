# AES Encryption with KMS

This repository contains Ansible roles and playbooks to encrypt and decrypt files with a key backed
by a KMS Customer Master Key.

## Pre-requisites

For the local machine executing the playbook, you will need to install at least
[Ansible 2.4](https://docs.ansible.com/ansible/latest/intro_installation.html).

For the machine where the encryption takes place (i.e. Ansible remote -- this can also be the local
machine), you will need the following installed:

- [AWS CLI](https://aws.amazon.com/cli/)
- OpenSSL

You will need the following for your AWS acocunt:

- AWS credentials on the machine performing the encryption. See [this](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) for more information.
- Customer Master Key (CMK) in Key Management Service (KMS)

## General Concepts

You might want to [read](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html) some
concepts for KMS from the AWS documentation.

In this repository, we encrypt data with the symmetric algorithm AES 256 CBC. The CMK in KMS acts
as a Key encryption key (KEK) and the master key for the AES key that will use to perform AES
encryption.

We ask KMS to generate the AES key (known as a "data key" in AWS documentation) and then encrypt it
with some
[encryption context](https://docs.aws.amazon.com/kms/latest/developerguide/encryption-context.html).

Each time we want to perform an encryption or decryption operation, we will ask KMS to decrypt the
encrypted data key first. We will store the decrypted key in memory for the duration of the
operation only.

## Playbooks

The playbooks contain full examples on how you might want to encrypt and decrypt files with a key
backed by KMS. Each playbook will generate a data key from KMS and then use the data key to perform
encryption and decryption. You can skip the various tasks using
[tags](https://docs.ansible.com/ansible/latest/playbooks_tags.html). You will also have to define
[variables](https://docs.ansible.com/ansible/latest/playbooks_variables.html)
either in the playbook themselves, or
[override](https://docs.ansible.com/ansible/latest/playbooks_variables.html#passing-variables-on-the-command-line)
on the command line while executing the playbooks.

### Tags

Use the tags to skip or only execute certain tasks.

- `generate_key`: KMS data key generation
- `encrypt`: Encrypt data
- `decrypt`: Decrypt data

### Directory Playbook

This playbook is defined in `directory.yml`. Files are encrypted from a template directory to a
destination encrypted directory, or decrypted from an encrypted directory to a destination
decrypted directory. Subdirectories are preserved.

The following variables are needed:

- `key_id`: ID of the CMK
- `cli_json`: Path to a [CLI JSON parameter](https://docs.aws.amazon.com/cli/latest/userguide/cli-using-param.html#cli-using-param-json) file. Use this to define the encryption context for your encrypted data key. You can use the default `cli.json` as a base.
- `key_output`: Path to output the encrypted data key to
- `template_dir`: Directory where your templates are.
- `encrypted_dir`: Directory to output encrypted files to. This will also be the input directory when decrypting.
- `decrypted_dir`: Directory to output the decrypted files to.
- `secrets_file`: A secret file to include. This file can be a file encrypted with [Ansible vault](https://docs.ansible.com/ansible/2.4/vault.html).

For example, to execute the playbook locally on your machine, you can do something like

```bash
ansible-playbook --inventory inventory --ask-vault-pass directory.yml
```

### List playbook

This playbook is defined in `list.yml`. This playbook encrypts and decrypts file based on a list
defined.

Each item in a list *must* contain the following keys:

- `template`: Path to the source template file
- `encrypted`: Path to the encrypted output or input
- `decrypted`: Path to the decrypted output

The following variables are needed:

- `key_id`: ID of the CMK
- `cli_json`: Path to a [CLI JSON parameter](https://docs.aws.amazon.com/cli/latest/userguide/cli-using-param.html#cli-using-param-json) file. Use this to define the encryption context for your encrypted data key. You can use the default `cli.json` as a base.
- `key_output`: Path to output the encrypted data key to
- `secrets_file`: A secret file to include. This file can be a file encrypted with [Ansible vault](https://docs.ansible.com/ansible/2.4/vault.html).
- `templates`: The list defined above.

For example, to execute the playbook locally on your machine, you can do something like

```bash
ansible-playbook --inventory inventory --ask-vault-pass list.yml
```

## Roles

The `roles` directory contains some common tasks that you might be able to reuse in your own
playbooks.

- `filters`: Contains some Jinja filters that are used by the rest of the roles and tasks.
- `kms-data-key`: Use AWS KMS to generate a new data key
- `kms-decrypt`: Use AWS KMS to decrypt some encrypted ciphertext.
- `aes-encrypt`: Encrypt plaintext with AES 256 CBC. The IV is appended as the final 16 bytes of the ciphertext.
- `aes-decrypt`: Decrypt ciphertext with AES 256 CBC. The IV is assumed to be the final 16 bytes of the ciphertext.

## Tasks

The `tasks` directory contains some reusable tasks that you can use in your own playbooks.

- `secrets`: This simply includes a variable file to your play.
- `generate_key`: Use KMS to generate a new data key.
- `read_key`: Read and optionally decrypt a KMS JSON output containing the data key for our encryption and decryption operations.
- `encrypt_directory` and `decrypt_directory`: Encrypt and decrypt files on a directory basis.
- `encrypt_list` and `decrypt_list`: Encrypt and decrypt files based on a provided list.
