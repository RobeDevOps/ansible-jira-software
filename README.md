# ansible-jira-software

```yaml
---
- import_tasks: install_postgres.yml
- import_tasks: create_service.yml
- import_tasks: create_files_configuration.yml
- import_tasks: create_and_configure_db.yml
- import_tasks: create_dbconfig_xml.yml
```
