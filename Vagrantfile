Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.hostname = "infra-ansible"
  config.vm.post_up_message = "README.md can have useful information sometimes!"
  config.vm.synced_folder ".", "/vagrant", type: "rsync",
    rsync__exclude: [".git/", ".terraform*"],
    rsync__args: ["--verbose", "--archive", "--delete", "-z", "--copy-links"]
  config.vm.provider "virtualbox" do |vb|
    vb.customize [ "modifyvm", :id, "--uartmode1", "disconnected" ] # to disable annoying logfile in Ubuntu VM
  end

  config.vm.provision "file", source: "files/locals/.bash_prompt_user", destination: "/tmp/.bash_prompt_user"
  config.vm.provision "file", source: "files/locals/.bash_prompt_root", destination: "/tmp/.bash_prompt_root"

# initializing script, executing only once when vagrant creates virtual machine from scratch
  config.vm.provision "shell", inline: <<~SHELL

# system settings and tunes
    set -v
    usermod -a -G adm vagrant
    sudo mv /tmp/.bash_prompt_user ~vagrant/.bash_prompt_user
    chown vagrant:vagrant .bash_prompt_user
    sudo mv /tmp/.bash_prompt_root /root/.bash_prompt_root
    chown root:root .bash_prompt_user
    sudo -u vagrant echo "[ -f ~/.bash_prompt_user ] && . ~/.bash_prompt_user" >> /home/vagrant/.bashrc
    echo "[ -f ~/.bash_prompt_root ] && . ~/.bash_prompt_root" >> /root/.bashrc
    su vagrant -c 'mkdir -p ~/.local/share'
    su vagrant -c 'mkdir -p ~/.config'

# let's enable time auto-sync
    timedatectl set-ntp on

# creating /bin/pass-login script to let ansible use keys from lastpass
    (
    cat <<EOF
      #! /bin/env bash
      set -e

      export LPASS_DISABLE_PINENTRY=1
      echo 'Enter environment name, please (prod, stage)'
      read TERR_ENV
      echo 'Enter LastPass username:'
      read TERR_LASTPASS_USERNAME
      echo 'Enter LastPass password:'
      read -s TERR_LASTPASS_PASSWORD
      echo 
      echo "\\$TERR_LASTPASS_PASSWORD" | lpass login \\$TERR_LASTPASS_USERNAME || {
      LASTPASSERROR=$?;
      echo "Login failed, error \\$LASTPASSERROR, please, try again!"; 
      exit \\$LASTPASSERROR;
      }

      export AWS_ACCESS_KEY_ID="\\$(lpass show awsterr/key/\\$TERR_ENV --username)"
      export AWS_SECRET_ACCESS_KEY="\\$(lpass show awsterr/key/\\$TERR_ENV --password)"
      export AWS_DEFAULT_REGION='us-east-2'

      mkdir -p $HOME/.aws

      echo "[default]" > \\$HOME/.aws/credentials
      echo "aws_access_key_id=\\$AWS_ACCESS_KEY_ID" >> \\$HOME/.aws/credentials
      echo "aws_secret_access_key=\\$AWS_SECRET_ACCESS_KEY" >> \\$HOME/.aws/credentials

      chmod 700 $HOME/.aws
      chmod 600 $HOME/.aws/credentials

EOF
    ) > /bin/pass-login && chmod +x /bin/pass-login
    
# ssh config
  echo >> /home/vagrant/.ssh/config
  echo "Host *" >> /home/vagrant/.ssh/config
  echo "StrictHostKeyChecking false" >> /home/vagrant/.ssh/config

# software packages upgrade and install

export DEBIAN_FRONTEND=noninteractive
export APT_LISTCHANGES_FRONTEND=none
apt-get -y --allow-releaseinfo-change update
apt-get -o Dpkg::Options::="--force-confnew" -fuyV --force-yes dist-upgrade
apt-get -y --purge autoremove
apt-get -y clean

apt-get -fuyV install neofetch mc net-tools traceroute mlocate links awscli lastpass-cli nmap ntpdate ruby

# Install terraform

curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
apt-get -y --allow-releaseinfo-change update
apt-get -fuyV --force-yes install terraform 
terraform -install-autocomplete

############## FZF (fuzzy command history) plugin install, not mandatory ##########################    
    echo "Installing fzf plugin for root and vagrant users..."
    cd
    git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
    cd ~/.fzf/
    ./install --all > /tmp/fzf_install_root.log 2>&1
    echo "set rtp+=~/.fzf" >> ~/.vimrc
    echo 'cd /vagrant' >> ~/.bashrc

    su vagrant -c "git clone --depth 1 https://github.com/junegunn/fzf.git /home/vagrant/.fzf"
    su vagrant -c "/home/vagrant/.fzf/install --all > /tmp/fzf_install_vagrant.log 2>&1"
    su vagrant -c "echo 'set rtp+=~/.fzf' >> /home/vagrant/.vimrc"
    su vagrant -c "echo 'cd /vagrant' >> /home/vagrant/.bashrc"
####################################################################################################


# EOF of the initializing script
SHELL
end   # end of the vagrant config.vm.provision section
