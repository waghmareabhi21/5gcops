# 5gcops
#Pre Install Environment Configuration

- hosts: sync_server
  become: yes
  gather_facts: yes
  remote_user: root
  tasks:
    - name: Searching chart-repo Image
      shell: "ls -l /repo/images/chartmuseum*|awk '{print $(NF)}'"
      register: 'filetocopy'

    - name: Sync Environment Setup
      local_action: file path=/tmp/images.cdm/ state=absent

    - name: Sync chart-repo images
      fetch :
        src: '{{ item }}'
        dest: /tmp/images.cdm/
        flat: yes
      with_items: '{{ filetocopy.stdout_lines }}'

- hosts: k8s_chart_repository
  become: yes
  gather_facts: yes
  remote_user: root
  tasks:
    - name: Registry Environment Setup
      file:
        path: /tmp/images.repo
        state: absent

    - name: Fetch chart-repo Image
      copy:
        src: /tmp/images.cdm/
        dest: /tmp/images.repo/

    - name: Create chart repo
      shell: |
        docker stop chartmuseum
        docker rm chartmuseum
        docker load < /tmp/images.repo/chartmuseum-latest.tar.gz
        mkdir ~/helm-chart-repo
        cd ~/helm-chart-repo
        docker run -dit -p 8080:8080  -v $(pwd)/charts:/charts -e DEBUG=true -e STORAGE=local -e STORAGE_LOCAL_ROOTDIR=/charts --name chartmuseum --restart always chartmuseum:latest
        chmod 777 -R ~/helm-chart-repo

