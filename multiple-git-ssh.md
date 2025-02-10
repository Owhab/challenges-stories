# Setting Up Multiple Git Accounts Using SSH Keys

Managing multiple Git accounts on the same machine can be challenging. This guide will help you set up multiple Git accounts using SSH keys.

## Step 1: Generate SSH Keys

First, generate a new SSH key for each Git account.

```sh
# Generate SSH key for personal account
ssh-keygen -t rsa -b 4096 -C "your_personal_email@example.com"
# Save the key as id_rsa_personal

# Generate SSH key for work account
ssh-keygen -t rsa -b 4096 -C "your_work_email@example.com"
# Save the key as id_rsa_work
```

## Step 2: Add SSH Keys to the SSH Agent

Start the SSH agent and add your SSH keys.

```sh
# Start the SSH agent
eval "$(ssh-agent -s)"

# Add personal SSH key
ssh-add ~/.ssh/id_rsa_personal

# Add work SSH key
ssh-add ~/.ssh/id_rsa_work
```

## Step 3: Add SSH Keys to GitHub/GitLab

Copy the SSH keys to your clipboard and add them to your GitHub/GitLab accounts.

```sh
# Copy personal SSH key
cat ~/.ssh/id_rsa_personal.pub

# Copy work SSH key
cat ~/.ssh/id_rsa_work.pub
```

## Step 4: Create SSH Config File

Create or edit the SSH config file to include your new keys.

```sh
nano ~/.ssh/config
```

Add the following configuration:

```sh
# Personal account
Host github.com-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_personal

# Work account
Host github.com-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_work
```

## Step 5: Clone Repositories Using SSH Config

When cloning repositories, use the host aliases defined in the SSH config file.

```sh
# Clone personal repository
git clone git@github.com-personal:username/repo.git

# Clone work repository
git clone git@github.com-work:username/repo.git
```

## Conclusion

You have successfully set up multiple Git accounts using SSH keys. You can now manage different Git accounts on the same machine without conflicts.
