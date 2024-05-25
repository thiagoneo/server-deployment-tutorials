### Instalação e configuração do servidor Samba no Rocky Linux 9

*29 de junho de 2023, por Thiago S. Ferreira*

Neste guia, criamos uma pasta compartilhada em um servidor Samba no sistema operacional Rocky Linux 9. O procedimento abaixo também deve funcionar em outras distribuições compatíveis, como Oracle Linux, AlmaLinux, CentOS Stream, e RHEL.

 1. Praticamente todos os comandos neste guia necessitam de permissão root, então, para evitar ter que usar o `sudo` em cada comando, fazer login como usuário `root`:

    ```
    su -
    ```
 2. Instalar os pacotes necessários:

    ```
    dnf install samba samba-client -y
    ```
 3. Habilitar o serviço do Samba:

    ```
    systemctl enable --now smb.service
    ```
 4. Permitir o Samba pelo firewall:

    ```
    firewall-cmd --permanent --add-service=samba
    firewall-cmd --reload
    ```
 5. Criar um usuário para acessar o compartilhamento Samba:

    ```
    adduser -M sambauser -s /sbin/nologin
    smbpasswd -a sambauser
    ```

    O comando `adduser` é para criar o usuário no sistema, neste caso, chamado de \`sambauser\`. O parâmetro -M é para evitar que seja criada a pasta do usuário, e `-s /sbin/nologin` é para definir o shell do usuário como `nologin`, dessa forma, não permitindo que esse usuário faça login no sistema (este usuário será utilizado exclusivamente para acesso ao compartilhamento Samba). Obs.: substituir `sambauser` pelo nome de usuário que será utilizado.
 6. Criar o diretório de compartilhamento:

    ```
    mkdir -p /data/shared
    ```

    Obs.: substitua `/data/shared` pelo caminho que será utilizado.
 7. Definir o usuário proprietário e as permissões do diretório criado:

    ```
    chown -R sambauser:sambauser /data/shared
    chmod -R 755 /data/shared
    ```

    Obs.: Novamente, altere `sambauser`pelo nome de usuário que você criou, e `/data/shared` pelo caminho da pasta respectiva, aqui e nas próximas ocorrências.
 8. Faça um backup do arquivo de configuração padrão do Samba:

    ```
    cp /etc/samba/smb.conf /etc/samba/smb.conf.bkp
    ```
 9. Utilize o nano ou o editor de sua preferência para editar o arquivo `/etc/samba/smb.conf` :

    ```
    nano /etc/samba/smb.conf
    ```
10. Insira este conteúdo no arquivo:

    ```
    [global]
    	workgroup = SAMBA
    	security = user
    	passdb backend = tdbsam
    
    [SharedFiles]
    	comment
    	path = /data/shared
    	guest ok = no
    	read only = no
    	writable = yes
    	valid users = sambauser
    	write list = sambauser
    	create mode = 0660
    	directory mode = 0770
    	nt acl support = no
    
    ```

    As opções definidas neste arquivo foram personalizadas para atender necessidades específicas, mas podem atender a maioria dos usuários que apenas desejam criar uma pasta compartilhada no Linux para acesso em outros computadores. A opção `valid users = scanners`, por exemplo, foi definida assim para permitir que apenas o usuário `sambauser` possa se autenticar e acessar o compartilhamento. As opções `create mode = 0660` e `directory mode = 0770` foram assim definidas para que os arquivos e pastas sejam criados com essas permissões, respectivamente. A opção `nt acl support = no` foi definida para impedir que os clientes alterem as permissões dos arquivos e pastas criados. Existem diversas opções que podem ser configuradas nesse arquivo, utilize-o como está, ou faça os ajustes que achar necessários. Para aprender mais sobre as opções disponíveis, consulte a documentação do Samba na internet, ou utilize o comando `man smb.conf`. Salve o arquivo com Ctrl + O, e saia com Ctrl + X (se estiver usando o `nano`).
11. Utilize o comando `testparm` para testar o arquivo.
12. Para que as alterações feitas no arquivo tenham efeito, reinicie o serviço do Samba com o comando:

    ```systemctl
    systemctl restart smb.service
    ```
13. O SELinux é uma camada de segurança que vem habilitada por padrão no Rocky Linux. Defina as permissões necessárias no SELinux sobre a pasta compartilhada, para que o Samba funcione enquanto mantém o SELinux habilitado.

    ```
    setsebool -P samba_export_all_rw on
    semanage fcontext -a -t samba_share_t "/data/shared(/.*)?"
    restorecon -rv /data/shared
    ```

Seguindo os passos acima, nosso servidor já deve estar funcionando. Aqui vai uma demonstração de como acessá-lo pelo Windows:

Pressione as teclas Ctrl + R, aparecerá a janela "Executar". Insira o seguinte:

```
\\ip-do-servidor
```

![Screenshot_20230630_004049.png](.attachments.3074/Screenshot_20230630_004049.png)

Pressione Enter, ou clique em "OK". Aparecerá uma janela de autenticação, onde você deverá inserir o usuário e a senha que você criou no passo 5.

![Screenshot_20230630_003954.png](.attachments.3074/Screenshot_20230630_003954.png)

Após se autenticar, o Windows explorer abrirá, e você poderá ver a pasta com o nome do compartilhamento que você definiu no arquivo `/etc/samba/smb.conf` , neste exemplo, definimos como `SharedFiles`.

![Screenshot_20230630_004215.png](.attachments.3074/Screenshot_20230630_004215.png)

Dentro dessa pasta você pode ler, criar, editar e excluir pastas e arquivos.

![Screenshot_20230630_004316.png](.attachments.3074/Screenshot_20230630_004316.png)

Fim.