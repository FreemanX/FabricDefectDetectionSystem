# Fabric Defect Detection System
Medical face masks are made of multiple(usually 3~5) layers of non woven fabric. During the production of non woven fabric, inevitablely, some dust or even insects may fall on the fabric.     
<img src="https://user-images.githubusercontent.com/9710644/168519343-fb1972d4-b85d-48ef-afc9-b6c14eacdc60.jpg" width="400">
<img src="https://user-images.githubusercontent.com/9710644/168519374-f53e3fc3-dea3-42ba-9310-395e8e34f68e.jpg" width="400">     
Although masks are manufactured in a dust-free environment, there are still chances that some masks are defective due to the contamination which comes with the raw material. For bacteria, the fabric is irradiated under UVC light on the automated production line right before it is used to manufacture masks. However, it is difficult to get rid of large specks. When multiple layers of fabric are stacked and become masks, it is extremely hard for the QC inspector to pick out the contaminated masks. To maintain the quality of masks, the factory put a worker at each production line by the raw material shelf and manually checks if there's dust or insect. Whenever the worker finds something unexpected, the whole production line must stop and wait for the worker cutting off the contaminated part and re-placing the fabric. You can image how tedious this job can be watching 3 ~ 4 layers of fabric for hours. On the other hand, because of Covid-19, the face mask market has extended to the general public. Conventional surgical masks are usually solid-colored like white, blue, or green. These masks are fine being used in the hospital but they are hard to be accepted by normal people, especially when people wearing masks on a daily basis. Therefore, many mask manufacturers introduce some fashion into their products by using fabrics with graphic patterns as the surface layer. As a result, it's nearly impossible for a person to find dust specks on a fast-moving fabric like that.   
## System design   
To sovle the problem, we, the R&D team, designed and implemented a fabric defects detection system leveraging the power of computer vision and machine learning.   
### Hardware design
<img width="600" src="https://user-images.githubusercontent.com/9710644/168565435-3361b8ec-63b2-4068-9055-f876afe1b5ac.png"> 
The figure above shows the main hardware components in the system. Baiscally, the fabric goes through the light box where there's a camera constantly watching the fabric.    
<img width="600" src="https://user-images.githubusercontent.com/9710644/168836666-1e240c84-7d76-4eae-a3af-af524c2a4292.png" align=center>
↑ Inside the light box

The realtime video feed is sent to a machine learning model for abnormality detection. If there's something unexpected, the Jetson nano will send a signal to a PCB designed by our hardware engineer. The PCB will then trigger the alarm and further forward the signal to the MCU(ESP8266). The MCU is capable of communicating with the production line so it knows the production rate and can tell the production line when to throw out the defective masks. Both MCU and NVIDIA Jetson Nano are connected to the internet so that realtime production status like how many good and bad masks have been produced, photos of contaminated fabric, system log, etc. to the cloud. Workers can interact with the system via a touch monitor connected to the Jetson Nano.   

### Software System Design   
An overview of the system is illustrated below:
![](https://user-images.githubusercontent.com/9710644/168803395-374bfb49-4ce2-4513-b09c-f96520661932.png)   
There are two parts in the system. One is the front-end software run on the nano for inferencing and the other is the back-end training server. We use Google Cloud, drive and sheets, as the intermediary connecting the front-end and the back-end.    
#### NVIDIA Jetson Nano
The fornt-end contains a graphical user interface that workers can make adjustments to the frames and detection sensitivity, check the live view, review the detection results and control other hardware components.   
<img width="600" src="https://user-images.githubusercontent.com/9710644/168835671-f7d47263-99ae-49c6-91ee-a1dfc9982801.jpeg">     
↑ Front-end GUI   
The application core serves as a control layer coordinating threads and different modules so that the system work together efficiently with limited resources(number of CPU cores, RAM size, computational power, etc.).    
Power manager will communicate with the UPS(Uninterruptible Power Supply). In case of power outage of the production line, it will notify the application core and corresponding event log will be recorded and upload to the cloud. Then the nano will turn itself off safely.    
GPIO Handler is responsible for sending and receiving signals from our custom-made PCB. For example, it can turn on and off the light box, start/stop the alarm, and communicate with the MCU.   
The frame processing engine contains the core functions. It will capture the video feed, pre-process each frame, detect abnormalities, render result images and send them to the user interface.   
Logger is a daemon thread that runs in the background recording the system status. Cloud handler is in charge of exchanging data between the nano and Google Cloud.   
#### Training Server    
The training server consists of two modules, data manager and task manager. Data manager will scan Google Drive periodically and download the latest photos uploaded from the production lines. The dataset generator will then convert these photos into trainable dataset. One of the challenges is that due to the nature of abnormality detetion, it is very difficult to collect enough training data(photos of fabrics with defects). Even if we deliberately smudge some fabrics, taking photos of them and labling the photos are still quite amount of work. Therefore, the dataset generator will generate synthetic data and lable them programmablely with photos of clean fabrics. The task manager will get notified when a new dataset is ready. The model trainer will train the model with the dataset and evaluate its performance afterwards. If the model performs better, the model deployer will send the model to a Jetson Nano at the server side where the model will be optimized specifically for the GPU on Jetson nano. Finally, the deployer will upload the optimized model to Google Drive.   
##### Continous Training
<img width="500" src="https://user-images.githubusercontent.com/9710644/168822620-524b06a3-2f25-4a4d-91fc-efa3e4d95866.png">     

A pretrained general model(model trained with both types of fabric, solid-colored and with different patterns) is used when the NVIDIA Jetson nano is deployed for the first time. Then the nano will collect photos of the fabric it sees and upload them to the cloud. At the server side, a dedicated model will be created for that nano and will be constantly fine-tuned with the latest photos.   
## Demo
### Detection results
#### Fabric with patterns   
<img width="700" src="https://user-images.githubusercontent.com/9710644/168971472-8dbe27b9-7850-4967-92da-4160a8e85c24.png">  

#### Fabric with solid color    
<img width="700" src="https://user-images.githubusercontent.com/9710644/168970869-94a9fc27-dfe0-4d40-8112-d21e888c9241.png">   
<img width="700" src="https://user-images.githubusercontent.com/9710644/168972319-b0a99c5e-4c34-4967-b0b1-1fca695c97c9.png">

### Automatically throw out contaminated masks 
<img width="700" src="https://user-images.githubusercontent.com/9710644/168974055-c6cb7b96-21c4-441e-9851-fd04fcb07055.JPG">   

[Video link](https://youtube.com/shorts/XUYxkAF2sp4?feature=share)   

