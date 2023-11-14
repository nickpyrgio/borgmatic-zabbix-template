# Borgmatic zabbix template
---
This is an active zabbix template for monitoring borgmatic tasks.

How does it work
---
- Import the template on the zabbix server
- Apply the template on the client on zabbix server
- Create a file containing the tasks should be written at the client.
    For example a file created as `/etc/ansible/facts.d/borgmatic_tasks.fact`:
    ```yaml
    {
        "data": [
            {
                "name": "borgmatic_task_name_1",
                "repository": "database",
                "server": "borg_server1.example.com",
                "user": "backup",
                "zabbix_item_key": "db_physical_1:ssh://backup@borg_server1.example.com/./database",
                "zabbix_nodata_warning_threshold": "28h"
            },
            {
                "name": "borgmatic_task_name_2",
                "repository": "database",
                "server": "borg_server2.example.com",
                "user": "backup",
                "zabbix_item_key": "borgmatic_task_name_2:ssh://backup@borg_server2.example.com/./database",
                "zabbix_nodata_warning_threshold": "28h"
            },
            {
                "name": "borgmatic_task_name_3",
                "repository": "binlogs",
                "server": "borg_server1.example.com",
                "user": "backup",
                "zabbix_item_key": "borgmatic_task_name_3:ssh://backup@borg_server1.example.com/./binlogs",
                "zabbix_nodata_warning_threshold": "1d"
            },
            {
                "name": "borgmatic_task_name_3",
                "repository": "binlogs",
                "server": "borg_server2.example.com",
                "user": "backup",
                "zabbix_item_key": "borgmatic_task_name_3:ssh://backup@borg_server2.example.com/./binlogs",
                "zabbix_nodata_warning_threshold": "1d"
            }
        ]
    }
    ```
- Run the following on the client
  ```bash
  cd /etc/zabbix && zabbix_sender -vv -c /etc/zabbix/zabbix_agent2.conf -k borgmatic.discovery.task -o "$(cat /etc/ansible/facts.d/borgmatic_tasks.fact)"
  ```
- Add the following line under the `after_backup`, `on_error`, `after_everything` and `before_everything` hook on the borgmatic task file:
  ```yaml
    after_backup:
        - echo "`date` - Finished backup."
        # - other custom commands ....
        # Change ${BORGMATIC_TASK_NAME} to the name of the borgmatic configuration file
        - cd /etc/zabbix;/usr/bin/zabbix_sender -c /etc/zabbix/zabbix_agent2.conf -k borgmatic.task.repository.status[${BORGMATIC_TASK_NAME}:{repository}] -o 1 || true
    on_error:
        - echo "`date` - Error while creating a backup."
        - cd /etc/zabbix;/usr/bin/zabbix_sender -c /etc/zabbix/zabbix_agent2.conf -k borgmatic.task.repository.status[${BORGMATIC_TASK_NAME}:{repository}] -o 0 || true
    # Optionally if you use scripts or command in before_everything or after_everything add
    before_everything:
        - myscript || (exit_code="$?";/usr/bin/jq -rc '.data[]|select(.name=="${BORGMATIC_TASK_NAME}").zabbix_item_key' /etc/ansible/facts.d/borgmatic_tasks.fact|while read zabbix_item_key;do cd /etc/zabbix; /usr/bin/zabbix_sender -c /etc/zabbix/zabbix_agent2.conf -k borgmatic.task.repository.status[$zabbix_item_key] -o 0 || true; done; exit "$exit_code")
    after_everything:
        - myscript || (exit_code="$?";/usr/bin/jq -rc '.data[]|select(.name=="${BORGMATIC_TASK_NAME}").zabbix_item_key' /etc/ansible/facts.d/borgmatic_tasks.fact|while read zabbix_item_key;do cd /etc/zabbix; /usr/bin/zabbix_sender -c /etc/zabbix/zabbix_agent2.conf -k borgmatic.task.repository.status[$zabbix_item_key] -o 0 || true; done; exit "$exit_code")
  ```

Every time the borgmatic task runs it will send a status flag to the zabbix server. 0 means failure, 1 success. If the server has not received an item for more than `zabbix_nodata_warning_threshold` an alert will be triggered.