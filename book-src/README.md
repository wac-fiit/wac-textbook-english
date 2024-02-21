# Introduction

_Development of web applications in the Cloud environment_ covers the area of software
application development intended for deployment in public data and computing centers,
commonly referred to as the _Cloud_. A specific element of developing such
applications is that the application is typically delivered directly from the data center
to end users, and the development team's work focuses on all aspects of the application:

* Development of the user interface for web browsers or mobile devices.
* Development of business rules and databases on the server side.
* Design and maintenance of suitable infrastructure for deploying the application.
* Deployment and monitoring of activities of the developed application.

In practice, this
development approach is referred to as _full-stack development_ or is also referred to as _DevOps_.

This text is intended as a materials for exercises in the subject with the same name for
students of engineering studies in information technology. The prerequisite for its study is
mastery of the basics of programming languages and operating systems.

The agile development approach, which is currently a typical way of organizing software
teams, focuses on obtaining feedback from real customers as quickly as possible.
This requires a development approach that prioritizes delivering a functional product first,
even if it is limited in terms of features or system robustness. Similarly, this textbook
is structured. In the early parts, it focuses on developing functional but limited web applications
and their deployment. The functionality of these applications is gradually expanded
with additional elements, whether in terms of functionality, data management, robustness,
or security and system maintenance. A significant part of the material is presented
in a form that corresponds to the ideas of agile development, development through microservices,
and in line with the rules expressed in the [_The twelve factor app_][twelves] manifesto.

The field of technologies and tools for developing applications for Cloud - data centers is broad
and rapidly evolving, and it is not possible to cover it completely in one textbook. The material
therefore focuses on currently well-known and commonly used technologies and tools,
which may become obsolete or replaced by newer technology in the future. The scope is
also limited by the time allocated for teaching this subject.

Exercises are continuously updated. They original version contain two versions of exercises:

* Angular and Asp.NET were the technologies chosen during the initial creation of the subject when Docker, Kubernetes, and containerized application techniques were just beginning. This part is excluded from the English translation.
* Currently, the materials also contain a second version of exercises based on the _micro front ends_ technology based on web components for the user interface and the use of the Go programming language for the backend application logic. Greater emphasis is also placed on using containerized application technology - _Docker_ - and their orchestration on the _Kubernetes_ platform.
![Technologies used in exercises](TechnologyStack_v2.png)

The subject is created and supported by the commercial entity
[Siemens Healthineers Development Center Slovakia][shsdc], which develops software products in the healthcare sector. The original text was prepared by Milan Unger as the subject lecturer, and the active contributors and co-authors of the new updated content are teacher fellows Peter Cagarda, Ivan Sládečka, and Michal Ševčík.
