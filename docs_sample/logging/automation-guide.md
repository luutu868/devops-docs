# Documentation Automation Guide for Software Projects

This guide outlines methods for automating software documentation generation without relying on GitHub or GitLab.

## Table of Contents
- [Self-hosted CI/CD Solutions](#self-hosted-cicd-solutions)
- [Custom Script Approach](#custom-script-approach)
- [Example Implementation with Jenkins](#example-implementation-with-jenkins)
- [Git Hooks Approach](#git-hooks-approach)
- [Documentation Tools Overview](#documentation-tools-overview)

## Self-hosted CI/CD Solutions

### 1. Jenkins
- Open-source automation server you can install on your own infrastructure
- Create pipelines that trigger on code commits
- Configure jobs to generate documentation and publish it
- Extensive plugin ecosystem for integration with various tools

### 2. TeamCity
- JetBrains' CI/CD solution you can self-host
- Supports documentation generation as build steps
- User-friendly interface with powerful configuration options
- Good integration with JetBrains IDEs

### 3. Bamboo (Atlassian)
- Enterprise-grade CI/CD that can run on your infrastructure
- Tight integration with other Atlassian products (Jira, Confluence)
- Built-in deployment capabilities

## Custom Script Approach

You can create custom scripts that:
1. Run on code commit hooks
2. Generate documentation using your preferred tools
3. Deploy to your documentation server

This approach gives you maximum flexibility but requires more manual setup and maintenance.

## Example Implementation with Jenkins

```groovy
pipeline {
    agent any
    
    stages {
        stage('Generate Code Documentation') {
            steps {
                // For Java projects
                sh 'javadoc -d ./docs/javadoc src/**/*.java'
                
                // Or for JavaScript
                sh 'jsdoc -d ./docs/jsdoc src/**/*.js'
                
                // Or for C++ with Doxygen
                sh 'doxygen Doxyfile'
            }
        }
        
        stage('Generate API Documentation') {
            steps {
                // For OpenAPI
                sh './gradlew generateOpenApiDocs' // or equivalent command
            }
        }
        
        stage('Generate Markdown Documentation') {
            steps {
                sh 'mkdocs build'
            }
        }
        
        stage('Publish Documentation') {
            steps {
                // Copy to documentation server
                sh 'rsync -av ./docs/ user@docserver:/var/www/html/docs/'
            }
        }
    }
}
```

## Git Hooks Approach

### Setting Up Git Hooks

1. Create a post-commit hook in your `.git/hooks/` directory:

```bash
# File: .git/hooks/post-commit
#!/bin/bash

# Generate code documentation
echo "Generating code documentation..."
javadoc -d ./docs/javadoc src/**/*.java

# Generate API documentation
echo "Generating API documentation..."
./gradlew generateOpenApiDocs

# Build markdown documentation
echo "Building markdown documentation..."
mkdocs build

# Deploy to documentation server
echo "Deploying documentation..."
rsync -av ./docs/ user@docserver:/var/www/html/docs/

echo "Documentation updated successfully!"
```

2. Make the script executable:
```bash
chmod +x .git/hooks/post-commit
```

3. Now documentation will be generated and deployed after each commit

### Making Hooks Shareable

To share hooks with your team, store them in the repository:

1. Store hooks in a folder within your repository:
```
/project-root
  /git-hooks
    post-commit
    pre-commit
    ...
```

2. Have developers symlink or copy these into their local `.git/hooks` directory
3. Or use Git's core.hooksPath configuration

## Documentation Tools Overview

### Code Documentation Generators
- **Javadoc (Java)**: Generates HTML documentation from Java source code comments
- **JSDoc (JavaScript)**: Creates HTML documentation from JavaScript comments
- **Doxygen**: Works with multiple languages (C++, C, Java, Python, etc.)
- **Sphinx (Python)**: Generates beautiful documentation from Python docstrings

### API Documentation
- **Swagger/OpenAPI**: Creates interactive API documentation
- **Tools like Springdoc** (Java/Spring) or **Express-OpenAPI** (Node.js) can generate these specs automatically

### Markdown Documentation
- **MkDocs**: Python-based static site generator for documentation
- **Jekyll**: Ruby-based static site generator
- **Docusaurus**: React-based documentation site generator
- **GitBook**: Documentation platform with a clean UI
