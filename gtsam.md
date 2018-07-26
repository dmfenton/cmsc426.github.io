---
layout: page
mathjax: true
title: Structure from Motion
permalink: /gtsam/
---
**This article is written by [Nitin J. Sanket](http://nitinjsanket.github.io).**

Table of Contents:




## Introduction
The SfM pipeline we talked about before relies on an optimization framework to do the bundle adjustment and you would've also observed that we also used `fminunc` or `lsqnonlin` for solving non-linear least squares in traingulation and PnP. This is not very efficient and there has to be a better way right? Guess what, there is! In 2012, Prof. Frank Dellaert published a hallmark paper called [GTSAM](https://smartech.gatech.edu/handle/1853/45226) for solving such problems which are commonly classified as **Pose Graph Optimization** and uses the concept of **Factor Graphs**. This method has been improved over the years and has periodically improved it's optimization methods with the latest literature. Though there are faster pose graph optimization frameworks like [**g2o**](https://github.com/RainerKuemmerle/g2o) or [**SLAM++**](https://sourceforge.net/p/slam-plus-plus/wiki/Home/), GTSAM by far is one of the easiest frameworks to setup, use and understand and hence will be the our choice to solve this project. 

Let us consider a toy example to understand how one would solve SfM/SLAM in GTSAM. For the purpose of brevity, I am assuming that you have followed the setup insturctions [here](https://borg.cc.gatech.edu/download.html) and setup GTSAM 3.2.1 on your computer with MATLAB compatibility. For any issues or help regarding the setup feel free to contact the TAs. Now, let's get back to the problem in hand. The problem is defined as follows. Our robot lives in a 2D world with certain number of unique landmarks  \\(N\\). **The origin in the world and the scale is arbitrary**, i.e., any point in the map can be chosen as the origin. A common choice is the **start of service** or the pose (position and orientation) of the robot at the beginning of the algorithm is chosen as the origin. The scale being arbitrary just means that all the distances are measured to some **relative scale**, i.e., metric scale is unknown. We only know that a landmark is twice as far as the other landmark and this could metrically mean that one landmark is 2m away or 2cm away, the information about this is not known. This is the case when we have only one camera. However, if a distance sensor like a SONAR or a LIDAR is used we would know the metric scale. Note that this doesn't change anything in the algorithm or the framework but merely changes the units in the computation. 

Now, let us say that each landmark is uniqely numbered so that we can distinguish between them. Let each landmark \\(\mathbf{l_k}\\) be represented by am \\(\mathbb{R}^2\\) vector which contains the \\(x\\) and \\(y\\) coordinate and is denoted by \\( \mathbf{l_k} = \begin{bmatrix} l_{k,x} & \_{k,y}\end{bmatrix}^T\\). The SfM/SLAM problem is to find the realtive pose of the landmarks and the pose of the robot when the robot is moving in the world. The key assumption here is that the landmarks are **stationary**. Dealing with non-stationary objects in the world is still an unsolved research topic. 

Mathematically, we want to get the best estimate of the landmark locations \\(\mathbf{l_k}\\) and robot pose at every time instant \\(\mathbf{x_t}\\) given the robot has a noisy sense of how it's moving (odometry or ego-motion) denoted by \\(\mathbf{o_{t}^{t+1}}\\) and the noisy measurements of the some landmark location(s) at each time instant \\(\mathbf{m_t}\\).  Let the state/pose of the robot at time \\(t\\) be defined as \\( \mathbf{x_t} = \begin{bmatrix} x_t & y_t & \theta_t \end{bmatrix}^T = \begin{bmatrix} x & y & \theta \end{bmatrix}^T_t  \\). Also, let us assume that the landmarks don't occlude each other and the camera has some field of view (say 120\\( ^\circ\\) for example).

Now, let's talk about the **measurement model**  or our model of the measurements from the camera sensor. This is the prior information of what we know about the camera sensor (field of view, resolution and so on) we are using and also the prior information about the landmark properties (a translucent landmark is bad for LIDAR, a featureless wall is bad for the camera and so on). 

- At the time \\(t\\) the robot is at pose \\( \mathbf{x_t} \\) and the robot observes \\(\mathbf{l_k}\\) landmarks where \\(k\\) denotes individual landmark IDs.
- The sensor is not perfect hence you see the landmark with a probability of \\( p_{obs} = 0.95\\), i.e., 95% of the time the sensor **sees** the landmark if it exists and 5% of the time the landmark doesnt see the landmark if it doesnt exist. Note that we are not considering the case of seeing a landmark if it doesn't exist.
- The measurement of the robot at time \\(t\\) with respect to landmark \\( \mathbf{l_k} \\) is given by \\( \mathbf{m_{t,k}} = \begin{bmatrix} m_{t,k,x} & m_{t,k,y}\end{bmatrix}^T\\). The measurement is a scaled noisy vector pointing from the robot's current pose to the landmark.
- \\( \mathbf{m_{t,k}} \\) is noisy with zero mean additive white gaussian noise. It can be modelled as drawn from \\( \mathbf{m_{t,k}} = \mathbf{\hat{m}_{t,k}}  + \mathcal{N}( 0, \Sigma_m)\\). Here, \\( \mathbf{\hat{m_{t,k}}} \\) is the noiseless measurement and \\( \Sigma_m\\) is the measurement noise. \\( \Sigma_m\\) generally can be found in the manufacturer datasheet of a particular sensor.

Now, let's talk about the **odometry model**  or our model of how the robot moves. This can come from simple measurements like a wheel encoder in a ground robot or motor speeds in a quadrotor/drone or fancy/complex measurements like a kinect or Visual Odometry (this involves computing feature matches, fundamental matrix, essential matrix, triangulation and  PnP). More about Visual Odometry later.  

- Odometry indicates the robot's estimates of it's own movement between two time instants \\(t \\) and \\(t + 1\\). This is denoted by \\( \mathbf{o_{t}^{t+1}}\\).
- The odometry estimates are noisy with zero mean additive white gaussian noise. It can be modelled as drawn from \\( \mathbf{o_{t}^{t+1}} = \mathbf{\hat{o}_{t}^{t+1}} + \mathcal{N}\left( 0, \Sigma_o \right)\\). Here, \\( \mathbf{\hat{o}_{t}^{t+1}} \\) is the noiseless measurement and \\( \Sigma_o\\) is the odometry noise. \\( \Sigma_o\\) generally a tuned parameter or estimated using a much more accurate localization setup like a [motion capture setup](https://www.vicon.com/). 

The **Simultaneous Localization and Mapping (SLAM)** or **Structure from Motion (SfM)** problem is defined as follows. Given initial pose \\(\mathbf{x_0}\\), odometry estimates \\(\mathbf{o_t^{t+1}}\\) and landmark measurements \\( \mathbf{m_t} \\) (all the measurements at time \\(t\\)), find the **Best estimate** of landmark locations \\( \mathbf{l_k}\\) and robot pose \\(\mathbf{x_t}\\) at every time instant. Observe that we said best estimate and not compute the perfect value. This is because, there is no way to find the perfect value unless we have \\(\infty\\) measurements (Refer to [**Central Limit Theroem**](https://en.wikipedia.org/wiki/Central_limit_theorem) for the reason). 
