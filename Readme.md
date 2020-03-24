## Ansible

### Crear un fichero de inventario: hosts-dev

```
# Por defecto /etc/ansible/hosts
# Podemos usar otro con -i <inventario>

# hosts-dev

[webservers]
3.228.178.191
3.232.218.105


[loadbalancer]
34.235.19.16

[local]
control ansible_connection=local
```

### Listar los hosts

```
$ ansible all -i hosts-dev --list-hosts
  hosts (4):
    control
    3.228.178.191
    3.232.218.105
    34.235.19.16
```

### Configuración de Ansible

La configuración se busca en el siguiente orden:

- Variable de entorno: ANSIBLE_CONFIG
- fichero ansible.cfg en el directorio actual
- fichero ~/.ansible.cfg (en el directorio home)
- Fichero /etc/ansible/ansible.cfg

Crear el fichero ansible.cfg

```
# ansible.cfg

[defaults]
inventory = ./hosts-env
```

```
NOTE: Windows WSL: So we cant have a configuration file which can be written by everyone, so you can set permission as 755 to /home/ansible-user/ansible directory and it should work like below example :

ansible-user@IND004513:~/ansible$ sudo chmod 755 /home/ansible-user/ansible/

ansible-user@IND004513:~/ansible$ ansible --list-hosts all
```

### Asignando nombre (Hostname DNS) a los hosts

```
# hosts-dev

[webservers]
webapp1 ansible_host=3.228.178.191
webapp2 ansible_host=3.232.218.105


[loadbalancer]
weblb ansible_host=34.235.19.16

[local]
control ansible_connection=local
```
Es posible utilizar patrones para seleccionar los hosts

```
$ ansible web* --list-hosts
  hosts (3):
    webapp1
    webapp2
    weblb
```

### Recursos

RESOURCES
[Install the Control Machine](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-the-control-machine)
[Working With Dynamic Inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html)
[Ansible Inventory File](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html)
[Ansible Configuration Settings](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#ansible-configuration-settings)
[Working with Patterns](https://docs.ansible.com/ansible/latest/user_guide/intro_patterns.html)


## Tareas Ansible

Las tareas en la forma de enviar comandos adhoc a las máquinas del inventario. La sintaxis:

```
$ ansible <options> <host-pattern>
```

```
$ ansible -m ping all
    ^      ^   ^   ^
    '      '   '   '
  command  '   '   '
         flag  '   '
        módulo '   '
               '   '
           nombre  '
           módulo  '
               Inventario 
```
```
$ ansible -a "uname" all
    ^      ^   ^      ^
    '      '   '      '
  ansible  '   '      '
         flag  '      '
       command '      '
               '      '
            comando   '
                      '
                  Inventario 
```
Ejemplos

```
# Ejemplo ejecución de un comando correcto (tb se puede usar -m shell -a "uname" all)
$ ansible -a "uname" all
control | SUCCESS | rc=0 >>
Linux

webapp2 | SUCCESS | rc=0 >>
Linux

weblb | SUCCESS | rc=0 >>
Linux

webapp1 | SUCCESS | rc=0 >>
Linux
```
```
# Ejemplo ejecución de un comando errónea (tb se puede usar -m command -a "false" \!local )
$ ansible -a "false" \!local
webapp1 | FAILED | rc=1 >>
non-zero return code

webapp2 | FAILED | rc=1 >>
non-zero return code

weblb | FAILED | rc=1 >>
non-zero return code
```


### Recursos

- Comando ansible: https://docs.ansible.com/ansible/latest/cli/ansible.html
- índice de módulos de ansible: https://docs.ansible.com/ansible/latest/modules/modules_by_category.html


## Playbooks
Documentación: https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html

Playbooks (libro de jugadas, lista de la compra o receta) es un script que incluye las intrucciones para gestionar la configuración de nuestros sistemas remotos

Permiten crear infraestructura como código y gestionar la configuración.
Las tareas se ejecutan en el orden establecido y pueden ejecutarse de forma síncrona o asíncrona

Utilizan sintasis Yaml
Están compuesto por uno o más plays, cuyo objetivo es realizar alguna tarea, mediant eun módulo ansible, en los host detallados en el inventario.
Por ejemplo, el comando:

```
$ ansible -m ping all
```
En un playbook sería:

```
# ping.yml
---
  - hosts: all
    task:
    - name: Ping a todos los servidores
      action: ping
```

Para ejecutar un playbook se utiliza el comando:

```
$ ansible-playbook ping.yml
```

### Tareas a realizar

- Gestión de paquetes: con el módulo correspondiente a la distribución linux instalada
- Configurar infraestructura: copiar (con el módulo copy) y sincronizar ficheros (con el módulo synchronize).
- Gestionar servicios: configurarlo con el módulo lineinfile e iniciarlos, pararlos o restaurarlo con el módulo service

- Gestión de paquetes

```
# yum-update.yml

---
  - hosts: webservers:loadbalancer
    become: true
    tasks:
      - name: Updating yum packages
        yum: name=* state=latest
```

- Gestionar servicios

```
# install-services.yml

---
  - hosts: loadbalancer
    become: true
    tasks:
      - name: Installing apache
        yum: name=httpd state=present
      - name: Ensure apache starts
        service: name=httpd state=started enabled=yes

  - hosts: webservers
    become: true
    tasks:
      - name: Installing services
        yum:
          name: 
            - httpd
            - php
          state: present
      - name: Ensure apache starts
        service: name=httpd state=started enabled=yes
```

- Instanlación y configuración de la APlicación

```
# setup-app.yml

---
  - hosts: webservers
    become: true
    tasks:
      - name: Upload application file
        copy:
          src: app/index.php
          dest: /var/www/html
          mode: 0755
      
      - name: Configure php.ini file
        lineinfile:
          path: /etc/php.ini
          regexp: ^short_open_tag
          line: 'short_open_tag=On'

      - name: restart apache
        service: name=httpd state=restarted
```

Es posible definir un handler para que apache se restaure si hay algún cambio en la configuración:

```
# setup-app.yml

---
  - hosts: webservers
    become: true
    tasks:
      - name: Upload application file
        copy:
          src: app/index.php
          dest: /var/www/html
          mode: 0755
      
      - name: Configure php.ini file
        lineinfile:
          path: /etc/php.ini
          regexp: ^short_open_tag
          line: 'short_open_tag=On'
        notify: restart apache
    handlers:
      - name: restart apache
        service: name=httpd state=restarted
```

- Configuración del balanceador

```
  # setup-lb.yml
---
  - hosts: loadbalancer
    become: true
    tasks:
      - name: Creating template
        template:
          src: config/lb-config.j2
          dest:  /etc/httpd/conf.d/lb.conf
          owner: bin
          group: wheel
          mode: 064
        notify: restart httpd
    
    handlers:
      - name: restart httpd
        service:
          name: httpd
          state: restarted
```

El fichero de configuración sería: 

```
ProxyRequests off
<Proxy balancer://webcluster >
    BalancerMember http://3.228.178.191
    BalancerMember http://3.232.218.105
    ProxySet lbmethod=byrequests
</Proxy>

# Optional
<Location /balancer-manager>
  SetHandler balancer-manager
</Location>

ProxyPass /balancer-manager !
ProxyPass / balancer://webcluster/
```

Se podría utilizar Jinja para el contenido
https://jinja.palletsprojects.com/en/2.11.x/

```
ProxyRequests off
<Proxy balancer://webcluster >
  {% for hosts in groups['webservers'] %}
    BalancerMember http://{{hostvars[hosts]['ansible_host']}}
  {% endfor %}
    ProxySet lbmethod=byrequests
</Proxy>

# Optional
<Location /balancer-manager>
  SetHandler balancer-manager
</Location>

ProxyPass /balancer-manager !
ProxyPass / balancer://webcluster/
```

### Englobar los playbooks en uno
https://docs.ansible.com/ansible/latest/modules/import_playbook_module.html

```
# all-playbooks.yml
---
  - import_playbook: yum-update.yml
  - import_playbook: install-services.yml
  - import_playbook: setup-app.yml
  - import_playbook: setup-lb.yml
```

### Crear un playbook para comprobar el estado de los servicios

```
# check-status.yml
---
  - hosts: webservers:loadbalancer
    become: true
    tasks:
      - name: Check status of apache
        command: service httpd status
```

