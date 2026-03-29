## Advanced Data Engineering with Snowflake

#### How to use this repo throughout the course:

To successfully follow along with the instructor during the course, you'll need to make use of the code in this repo. To follow along, you can either:

* Keep the URL to this repo handy, so that you can easily find and use any code referenced by the instructor during the course

* Clone the repo to your local computing environment (required)

> **Note:** There are a couple of exercises that make use of Snowflake's command line interface, Snowflake CLI. To successfully follow along during those exercises, you'll need to have the repo cloned to your local computing environment, so that the Snowflake CLI can make use of files and code within this repo.

#### How to clone the repo to your local computing environment:

1. Fork the repo to create a copy associated with your GitHub Account: https://github.com/Snowflake-Labs/advanced-data-engineering-snowflake/fork

2. Clone your fork:

```bash
git clone https://github.com/<your-GitHub-user-name>/modern-data-engineering-snowflake.git
```

Where `<your-GitHub-user-name>` is replaced by your GitHub user name. This workflow is covered in the course.

You can then open the repo in your preferred code editor. Throughout the course, the instructor will use Visual Studio Code as the code editor.

#### How to navigate this repo

All of the code that you need to successfully complete the course is within this repo. Each folder in this repo corresponds to a module in the online course.

* **module-1** – Corresponds to "Module 1: DevOps with Snowflake" in the course.

* **module-2** – Corresponds to "Module 2: Observability with Snowflake" in the course.



### Module 1 - Steps followed

```text
 git remote set-url origin https://github.com/nilanka-weeraman/advanced-data-engineering-snowflake

------------------------------------------------------
-- correct the marketplace fetched data table names 
------------------------------------------------------
```

#### create git PAT secret, git api integrtion & setup repository on Snowflake IDE

```text

USE ROLE accountadmin;
CREATE DATABASE course_repo;
USE SCHEMA public;   
-- Create credentials
CREATE OR REPLACE SECRET course_repo.public.github_pat
  TYPE = password
  USERNAME = 'nilanka-weeraman'
  PASSWORD = 'xxxxx';

-- Create the API integration
CREATE OR REPLACE API INTEGRATION git_api_integration
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/nilanka-weeraman') -- URL to your GitHub profile
  ALLOWED_AUTHENTICATION_SECRETS = (github_pat)
  ENABLED = TRUE;

-- Create the git repository object
CREATE OR REPLACE GIT REPOSITORY course_repo.public.advanced_data_engineering_snowflake
  API_INTEGRATION = git_api_integration -- Name of the API integration defined above
  ORIGIN = 'https://github.com/nilanka-weeraman/advanced-data-engineering-snowflake.git' -- Insert URL of forked repo
  GIT_CREDENTIALS =  course_repo.public.github_pat;

-- List the git repositories
SHOW GIT REPOSITORIES;
```

### create staging & prod database objects, tables, views, procs, streams using script in git

```text
 snow git fetch advanced_data_engineering_snowflake --database=course_repo --schema=Public


 snow git execute @advanced_data_engineering_snowflake/branches/main/module-1/hamburg_weather/pipeline/objects/ -D "env='STAGING'"    
   --database=course_repo --schema=Public
 snow git execute @advanced_data_engineering_snowflake/branches/main/module-1/hamburg_weather/pipeline/objects/ -D "env='PROD'"       
   --database=course_repo --schema=Public
```

#### fix the issue in the load_tasty_bytes.sql file for DDL command    

```text
 git fetch
 git switch fix-missing-data-2
 git branch
 git status
 git add -p
 git commit -m "fix missing data"
 git push origin fix-missing-data-2
```

#### create a pull request in git from 'fix-missing-data-2' to 'statging'

