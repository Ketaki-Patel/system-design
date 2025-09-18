### system-design
Jotting down nitty gritty details of system-design

I recently implemented an alerting system with two microservices—Vitals Service and Alerts Service—using Spring WebFlux and Project Reactor.

Previously, I’ve worked extensively on notification services for healthcare systems and even built DiveAlert from scratch for a diving platform. Drawing from these experiences, I’ve decided to start a series of practical articles on commonly used system design patterns—covering real-world approaches I’ve repeatedly applied.

To kick things off, I’ll start with article related to real-time messaging system, goal is to publish this eventually on medium or linkedIn

Stay tuned for the first deep dive!

### Real Time Messaging System

#### Why I chose this as my first deep dive?
- This is such a versatile design requirement(i.e real time messaging) which makes backbone for very popular system design architectures like
    - real time chat service (e.g whatsapp, signal, telegram etc)
    - broad casting platfor(e.g twitter)
    - notifications platform for social media network(i.e facebook, instagram), you can extends this design for SMS, Email and push notifications


<img width="890" height="762" alt="image" src="https://github.com/user-attachments/assets/16c62b9b-dfc5-42ec-97aa-beea76b05e11" />
  
1. Real Time Messaging System 
2. Notifiction Service plaform
   More to come 


