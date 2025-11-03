### typ-e
```mysql
CREATE USER IF NOT EXISTS 'graphql_report_ro'@'%' IDENTIFIED BY '';

-- Grant SELECT permissions on các bảng đã chỉ định cho 'graphql_report_ro'@'%'
GRANT SELECT ON `tinybots`.`robot_account` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `tinybots`.`user_robot` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `tinybots`.`user_account` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `tinybots`.`robot_online_status` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `tinybots`.`robot_online_status_history` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `tinybots`.`script_reference` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `tinybots`.`script_execution` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `tinybots`.`script_version` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `tinybots`.`script_category` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `tinybots`.`robot_schema` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `tinybots`.`task_schedule` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `tinybots`.`robot_profile` TO 'graphql_report_ro'@'%';
```


### dashboard

```mysql
CREATE USER IF NOT EXISTS 'graphql_report_ro'@'%' IDENTIFIED BY '';

-- Cấp quyền SELECT trên các bảng của database 'dashboard' cho 'graphql_report_ro'@'%'
GRANT SELECT ON `dashboard`.`taas_order` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `dashboard`.`dashboard_relation` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `dashboard`.`taas_subscription` TO 'graphql_report_ro'@'%';
GRANT SELECT ON `dashboard`.`dashboard_robot` TO 'graphql_report_ro'@'%';
```