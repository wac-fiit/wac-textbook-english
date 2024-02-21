# Summary

We have demonstrated the basic process of creating a single-page web application using the web components technique and micro-front-end integration. We used the Stencil JS library and a micro-front-end controller created for the needs of this project.

The created application is significantly simplified, and for the sake of time-saving, we did not show many techniques that can be crucial in real projects. In addition to the Stencil JS library, other libraries can be used for creating web components. It is recommended to explore libraries such as [Lit]. However, web component creation is also supported in other popular libraries such as [Angular], [React], or [Vue].

We did not address the management of the application state in this exercise, which can be crucial in larger application systems. We recommend studying one of the implementations of the [Redux] design pattern for this purpose. We also did not address the issue of dependency inversion between system classes - known as _Inversion of Control_, which can complicate application testing and slow down development during the future evolution of the system. The [Inversify] library is becoming increasingly popular for this purpose. In any case, we recommend following the principle of _micro components_, creating the smallest possible components responsible for a specific functionality and combining them into larger entities - also web components - but with simple application logic. With this approach, the use of the Inversify library or Redux is not necessary, as the risk of excessive complexity of application building blocks is reduced.

If you do not plan to use the micro-front-end technique and are more interested in creating a Single Page application with a library that addresses various aspects of the architecture and maintenance of modern team-developed applications, we recommend trying the [Angular] library.

