# ELK-Setup

<pre><code>
sudo apt-get update
sudo apt-get upgrade -y
sudo hostnamectl set-hostname elkServer
sudo timedatectl set-timezone Asia/Ho_Chi_Minh</code></pre>
---------------------------
# Cài đặt java 8 jdk
<pre><code>sudo apt install -y openjdk-8-jdk</code></pre>
----------------
# Cài đặt elasticsearch
<pre><code>
sudo apt-get install apt-transport-https
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt-get update && sudo apt-get install elasticsearch
</code></pre>

#Cấu hình elasticsearch
<pre><code>
sudo nano /etc/elasticsearch/elasticsearch.yml
</code></pre>
# Comment cái này #network.host: 192.168.0.1 và thêm dòng này bên dưới network.host
<pre><code>network.bind_host: ["127.0.0.1", "địa_chỉ_ip_elk_server"]</code></pre>
# Thêm cuối dòng discovery.type: single-node
<pre><code>sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch</code></pre>

# Kiểm tra trạng thái elasticsearch
<pre><code>sudo systemctl status elasticsearch</code></pre>

# Kiểm tra thông tin của elasticsearch

<pre><code>sudo apt-get install -y curl
curl -X GET "localhost:9200"</code></pre>

# Kiểm tra các port, ip, giao thức
<pre><code>apt install net-tools
netstat –plntu </code></pre>
------------------
# Cài đặt Kibana
<pre><code>echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt-get update && sudo apt-get install -y kibana</code></pre>

# Cấu hình kibana

<pre><code>sudo nano /etc/kibana/kibana.yml</code></pre>

# Sửa server.host 0.0.0.0
<pre><code>sudo systemctl enable kibana
sudo systemctl start kibana</code></pre>

# Kiểm tra kibana
<pre><code>netstat –plntu </code></pre>
-----------------
# Cài đặt Logstash

<pre><code>echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt-get update && sudo apt-get install -y logstash</code></pre>

# Tạo 1 file có tên là 01-logstash.conf
<pre><code>nano /etc/logstash/conf.d/01-logstash.conf</code></pre>
# Cấu hình trong file đấy bằng đoạn code sau
<pre><code>
input {
  beats {	
    port => 5044
  }
}
filter {
  if [fileset][module] == "system" {
    if [fileset][name] == "auth" {
      grok {
        match => { "message" => ["%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} sshd(?:\[%{POSINT:[system][auth][pid]}\])?: %{DATA:[system][auth][ssh][event]} %{DATA:[system][auth][ssh][method]} for (invalid user )?%{DATA:[system][auth][user]} from %{IPORHOST:[system][auth][ssh][ip]} port %{NUMBER:[system][auth][ssh][port]} ssh2(: %{GREEDYDATA:[system][auth][ssh][signature]})?",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} sshd(?:\[%{POSINT:[system][auth][pid]}\])?: %{DATA:[system][auth][ssh][event]} user %{DATA:[system][auth][user]} from %{IPORHOST:[system][auth][ssh][ip]}",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} sshd(?:\[%{POSINT:[system][auth][pid]}\])?: Did not receive identification string from %{IPORHOST:[system][auth][ssh][dropped_ip]}",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} sudo(?:\[%{POSINT:[system][auth][pid]}\])?: \s*%{DATA:[system][auth][user]} :( %{DATA:[system][auth][sudo][error]} ;)? TTY=%{DATA:[system][auth][sudo][tty]} ; PWD=%{DATA:[system][auth][sudo][pwd]} ; USER=%{DATA:[system][auth][sudo][user]} ; COMMAND=%{GREEDYDATA:[system][auth][sudo][command]}",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} groupadd(?:\[%{POSINT:[system][auth][pid]}\])?: new group: name=%{DATA:system.auth.groupadd.name}, GID=%{NUMBER:system.auth.groupadd.gid}",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} useradd(?:\[%{POSINT:[system][auth][pid]}\])?: new user: name=%{DATA:[system][auth][user][add][name]}, UID=%{NUMBER:[system][auth][user][add][uid]}, GID=%{NUMBER:[system][auth][user][add][gid]}, home=%{DATA:[system][auth][user][add][home]}, shell=%{DATA:[system][auth][user][add][shell]}$",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} %{DATA:[system][auth][program]}(?:\[%{POSINT:[system][auth][pid]}\])?: %{GREEDYMULTILINE:[system][auth][message]}"] }
        pattern_definitions => {
          "GREEDYMULTILINE"=> "(.|\n)*"
        }
        remove_field => "message"
      }
      date {
        match => [ "[system][auth][timestamp]", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
      }
      geoip {
        source => "[system][auth][ssh][ip]"
        target => "[system][auth][ssh][geoip]"
      }
    }
    else if [fileset][name] == "syslog" {
      grok {
        match => { "message" => ["%{SYSLOGTIMESTAMP:[system][syslog][timestamp]} %{SYSLOGHOST:[system][syslog][hostname]} %{DATA:[system][syslog][program]}(?:\[%{POSINT:[system][syslog][pid]}\])?: %{GREEDYMULTILINE:[system][syslog][message]}"] }
        pattern_definitions => { "GREEDYMULTILINE" => "(.|\n)*" }
        remove_field => "message"
      }
      date {
        match => [ "[system][syslog][timestamp]", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
      }
    }
  }
}
output {
  elasticsearch {
    hosts => ["localhost:9200"]
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
</code></pre>

# Tiến hành enable logstash
<pre><code>sudo systemctl enable logstash</code></pre>
------------------------------
Đến đây, ta đã cài đặt và cấu hình xong elk_stack
_________________________________
Tiếp đến, ta cài đặt beats để gửi log lên elk server. Việc này tùy các bạn lựa chọn
mô hình elk. Dưới đây mình chọn cài filebeat ở máy client để gửi log từ client về elk server
----------------------------------

# Tiếp theo cấu hình filebeat ở client1

<pre><code>wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt-get update
sudo apt-get install -y filebeat
#truy cập vào file config để cấu hình
sudo nano /etc/filebeat/filebeat.yml</code></pre>

# Sửa enable phần filebeat input thành true, path: - /var/log/secure và - /var/log/messages - /var/log/suricata/
test đầu ra của filebeat
<pre><code>filebeat test output
systemctl enable filebeat 
filebeat modules enable system
filebeat modules enable suricata</code></pre>

# Sau đó ta load dashboard và index template cho kibana với lệnh
<pre><code>filebeat setup –e</code></pre>
# Quay lại elkserver để test
<pre><code>curl -X GET 'http://localhost:9200/filebeat-*/_search?pretty'</code></pre>
