#### Inventory file
Đầu tiên chúng ta sẽ tạo một file ``inventory.ini`` thay vì sử dụng file mặc định tại ``/etc/ansible/hosts``. Nội dung của file ``inventory.ini`` đơn giản như sau : 

```
[app]
10.0.10.138
```

Tiếp theo ta tạo một playbook mới tên là ``install-laravel.yml`` có nội dung như sau : 

```
---
- hosts: app
```

#### Variable file, pre-tasks and handlers

Ta có thể hiểu **variable** như là một biến môi trường ( environment variables ). Các biến này có thể được viết trực tiếp trong file đó hoặc tách ra một file riêng biệt. Ở đây chúng ta sẽ dùng ``var_files`` để chỉ định danh sách các file chứa các biến mà playbook đó sử dụng. Variable file cũng sẽ được viết dưới dạng YAML

Trong ví dụ của chúng ta, ``install-laravel.yml`` playbook sẽ sử dụng một variable file duy nhất - ``vars.yml``. Ta sẽ phải tạo file đó nếu nó chưa tồn tại và thêm các dòng này vào playbook: 

```
---
- hosts: app
  vars_files:
    - vars.yml
```

*Danh sách các biến có thể được viết trực tiếp trong playbook sử dụng ``var`` key thay vì ``vars_files``. Tuy nhiên việc tách các biến ra một file riêng biệt sẽ giúp playbook gọn gàng hơn và các biến sẽ được tập trung về một chỗ, dễ dàng cho những thay đổi sau này*

##### Pre-tasks 
Ansible cho phép chúng ta thực hiện các tasks trước và sau danh sách các tasks chính trong playbook thông qua ``pre_tasks`` và ``post_tasks``. 

Ở đây chúng ta cần cập nhật ``apt cache`` trước khi chạy các task chính trong playbook để đảm bảo thông tin của các packages đã được cập nhật : 

```
pre_tasks:
  - name: Update packages information.
    apt:
      update_cache: yes
      cache_valid_time:  3600
```

##### Handlers 

Đây là một loại task đặc biệt trong Ansible, các tasks này chỉ chạy khi được chỉ định ở cuối các task chính bằng cách sử dụng ``notify`` option. Các handlers sẽ chỉ chạy khi các tasks gọi đến chúng. Ta sẽ sử dụng nginx trong ví dụ này và việc load lại cấu hình của nginx sẽ diễn ra khá thường xuyên sau khi thay đổi cấu hình của nó. Vì vậy ta một một handlers để sử dụng : 

```
handlers:
  - name: reload Nginx
    service:
      name: nginx
      state: reloaded
```
Sử dụng handlers này ở cuối một task như sau : ``notify: reload Nginx``

### Basic configurations

Bây giờ ta sẽ định nghĩa các task chính của playbook. Đầu tiên chúng ta cần cài đặt các phần mềm cần thiết.

Trước khi cài đặt các phần mềm chính, có một số gói phụ thuộc cần phải cài đặt trước, cụ thể là ``python3-apt`` và ``python3-pycurl``. Ansible cung cấp cho chúng ta các công cụ để cài đặt một danh sách các phần mềm nếu cần thiết. Ở đây chúng ta sẽ sử dụng ``with_items``. Cụ thể chúng ta sẽ có một task như sau : 

```
- name: Install software for apt repository management.
  apt:
    name: "{{ item }}"
    state: present
  with_items:
    - python3-apt
    - python3-pycurl
```

Hiểu đơn giản ở đây chúng ta có một mảng ``items`` được định nghĩa thông qua ``with_items`` option.

Tiếp theo là cài đặt PHP, PHP extensions, MySQL và NGINX, chúng ta sẽ sử dụng ``with_items`` để thực hiện công việc này. Tuy nhiên chúng ta cần thêm Ubuntu PPA để cài đặt phiên bản mới hơn của PHP. Ở đây PPA là ``ppa:ondrej/php``. Việc thêm PPA sẽ được thực hiện thông qua ``apt-repository`` module như sau : 

```
- name: Add ondrej repository for latest version of PHP.
  apt_repository:
    repo: ppa:ondrej/php
    update_cache: yes
```

Bây giờ sẽ cài đặt các phần mềm sử dụng task sau : 

```
- name: Install NGINX, MySQL, PHP, PHP extensions and other dependencies.
  apt:
    name: "{{ item }}"
    state: present
  with_items:
    - git
    - curl
    - unzip
    - openssl
    - nginx
    - php7.1-common
    - php7.1-cli
    - php7.1-dev
    - php7.1-gd
    - php7.1-curl
    - php7.1-json
    - php7.1-opcache
    - php7.1-xml
    - php7.1-mbstring
    - php7.1-pdo
    - php7.1-mysql
    - php7.1-zip
    - php7.1-fpm
    - php-apcu
    - libpcre3-dev
    - python-mysqldb
    - mysql-server
```

Khởi động một số service quan trọng như ``nginx`` và ``mysql`` : 

```
- name: Start NGINX, MySQL services.
  service:
    name: "{{ item }}"
    state: started
    enabled: yes
  with_items:
    - nginx
    - mysql
```

### Configure MySQL

Cấu hình cho MySQL bao gồm các bước sau : 

- Xóa test database mà MySQL cung cấp sẵn
- Tạo mới một database cho ứng dụng
- Tạo mới một user

Để xóa database ``test`` mà MySQL cung cấp sẵn, chúng ta sẽ sử dụng ``mysql_db`` module với tên của database là ``test`` và ``state`` sẽ là ``absent`` : 

```
- name: Remove the MySQL test database.
  mysql_db:
    db: test
    state: absent
```


