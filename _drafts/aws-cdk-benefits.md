I work in a mid-sized company with a fairly small development team. In these cases, there are no different SRE or infra teams. Everything is done by the developers.  
Maybe shifting left can be easily done when your company is frugal with engineering🌚, where developers cry all day with VSCode and AWS Console open in their monitors

Until now, we managed most of our infra manually from the AWS Console (our team does not use many cloud-native services, we also provide on-premise installations, so having a simple setup is a necessity for us). We have been reaching the limits though, its just too painful at this point to keep going.  
The next step in these cases is to use [CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) for managing our resources. I've not used cloudformation before (I did use K8s a bit in my career in the beginning), so my opinions are not well-formed right now. The problems that made me really hesitant to use CloudFormation are
- I don't like the idea of YML too much, I do however, see value in keeping **your infra code as explicit and as simple as you can**
- Its too verbose, like really verbose
- Its hard to compose or reuse definitions nat
