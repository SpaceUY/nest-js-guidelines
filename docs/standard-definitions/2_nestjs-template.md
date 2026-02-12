---
title: NestJS Template
parent: Company’s Standard Technology Definitions
layout: default
nav_order: 2
---

# NestJS Template

We provide a [NestJS Template](https://github.com/SpaceUY/NestJS-Template) as a starting point for backend projects. The main goal of this template is to offer a development guide and pre-built implementations of common features used across multiple projects. By leveraging the [planetary tool](https://github.com/SpaceUY/planetary), these implementations can be easily exported and integrated into new backend projects.

## How to Start a New NestJS Backend?

If you need to start a backend with NestJS from scratch:

1. **Create a new project:**
   - Use the official NestJS CLI commands to initialize a new project. Follow the [NestJS first steps guide](https://docs.nestjs.com/first-steps?utm_source=chatgpt.com) for detailed instructions.
2. **Reference the official template and guidelines:**
   - Use the [NestJS Template](https://github.com/SpaceUY/NestJS-Template) and your team's guidelines to organize your folders and structure your codebase. For more details, see the [Project Structure section](../architecture/2_project-structure.md).
3. **Copy adaptable modules from the template:**
   - Take only the folders or modules you need from the template. You can do this manually via copy/paste or by using the [planetary tool](https://github.com/SpaceUY/planetary) for an automated approach.
4. **Enjoy developing on the dark side!**

> **WARNING:**
> Do **not** start a backend project directly from the template repository. The template may not always be fully up to date. Instead, use it as a reference or to extract adaptable modules to accelerate your development process.

**Adapter Pattern**

The template makes use of the Adapter Pattern to provide flexibility and extensibility for different integrations. This is achieved by defining a common abstract class for each module (such as Email, Cloud Storage, or Push Notifications), and then implementing provider-specific subclasses that extend this abstract class. Each subclass handles the integration with a particular provider (e.g., SendGrid, Resend, AWS SES for Email), allowing you to switch or add providers with minimal changes to the rest of your codebase.

The adapter pattern ensures a consistent interface for each module, making it easy to use and extend integrations as needed across different projects.

**Available adaptable modules:**
- **Email:** Includes integration with SendGrid. [email module](https://github.com/SpaceUY/NestJS-Template/pull/14)
- **Cloud Storage:** Includes integration with S3 ([cloud-storage module](https://github.com/SpaceUY/NestJS-Template/tree/master/src/cloud-storage)).
- **Push Notifications:** Includes integration with Expo ([push-notification module](https://github.com/SpaceUY/NestJS-Template/tree/master/src/push-notification)).

The template is currently being updated to align with the latest company standards. We encourage contributions! To participate, join the NestJS chapter calls to suggest new implementations or to take ownership of a feature you would like to add.

> **Note:** Before using the template to start a new project, please ensure it is up to date with the latest versions, standards and features.

**Coming soon:**
- Auth0 integration (In Pull Request)
- TypeORM implementation
- Resend and AWS SES integration for emails (In Pull Request)
