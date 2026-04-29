github:
**项目**
	obsidian-vault:git@github.com:ZWKAI/obsidian-vault.git
**维护远程仓库**
	添加远程仓库地址：
		git remote add origin git@github.com:ZWKAI/obsidian-vault.git
	修改远程仓库地址：
		git remote set-url origin git@github.com:ZWKAI/obsidian-vault.git



**秘钥配置**：
	创建秘钥：
		ssh-keygen -t ed25519 -C "892008350@qq.com" -f ~/.ssh/id_rsa_github
	添加至agent
		ssh-add --apple-use-keychain ~/.ssh/id_rsa_github
	复制公钥：pbcopy < ~/.ssh/id_rsa_github.pub

	添加至git
		Settings --> SSH and GPG keys --> New SSH key 按钮
