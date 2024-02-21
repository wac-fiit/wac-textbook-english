# Exercise 2: Creating a web service using the Go language

## <a name="goal"></a>Exercise Goal

The goal of the exercise is to create a web RESTful service for the waiting room management application described
in the previous exercise. Basic functionality:

* The application has two users: Patient and Clinic Nurse
* Patient
  * As a patient, I want to enter my personal identification number (or patient number) upon arrival at the waiting room
  and have the system queue me and inform me of my position and an approximate time for my examination.
  * As a patient with an acute illness - injury, high temperature, pregnancy - and eligibility for priority treatment,
  I want to enter my personal identification number (or patient number) upon arrival at the waiting room and have the system
  queue me in a list with priority treatment.
  * As a patient waiting in the clinic, I want to have a visual overview of my current queue position.
* Clinic Nurse
  * As a clinic nurse, I want an overview of the number and identity of waiting patients and the next patient in line.
  * As a clinic nurse, I want to know how many and which patients are waiting for medical examination, waiting for priority treatment,
  and waiting for administrative tasks to be processed.
  * As a clinic nurse, in the case of assessing a serious patient condition, I want the ability to change the queue order.

In addition, there are other users: "Hospital Administrator" and "Application Developer."
Application functionality is defined as follows:

* Hospital Administrator (_will not be implemented during the exercise_)
  * As a hospital administrator, I want to get a list of all clinics in the hospital,
    the current number of waiting patients at each clinic, and the average waiting time
    over the last ten days.
  * As a hospital administrator, I want to create a new clinic.
  * As a hospital administrator, I want to be able to set basic clinic parameters,
    such as its name, door number, attending doctors, opening hours, and so on.
* Application Developer
  * As a software system developer, I want to know what functionality the web service
  provides and what parameters it requires, or what return values I can expect.
  * As an application developer, I want the ability to easily test and verify
  the functionality of the web service.

Technical constraints:

* The application is created in the [Go] language using the [Gin Web Framework][gin].
* The application interface is described in [OpenAPI] format.
* The waiting queue status and patient information are stored in a No-SQL database,
  which will be accessed using the [MongoDB] library.

## <a name="preparation"></a>Preparation for Exercise

* Created the application according to the instructions in the exercise
  [_Web application using the Stencil JS library_](../01.Web-Components/000-README.md)
* Installed the Go programming language [Go](https://go.dev/doc/install)
* Installed the [Postman] application.
* Familiarized yourself with the Go language, e.g., [GOLANGBOT.COM](https://golangbot.com/learn-golang-series/)
* Familiarized yourself with the [YAML] language (https://yaml.org/)
