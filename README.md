# Computer-Vision-Based-Customer-Traffic-Monitoring-483-Project
Project for our 483 Machine Learning class.

Project Title: Computer Vision-Based Customer Traffic Monitoring 
Members: Cardin Phan [Undergraduate], Avery Miclea [Undergraduate]

Introduction.
Retail businesses can be optimized by using machine learning analysis to understand customer behaviors. By having a machine learn how customers move through a store, which areas receive the most traffic, and what times of day are the busiest we might be able to further optimize businesses. This information may require manual observation, sales data, or dedicated sensors, but instead we can use machine learning.

Motivation.
We were both interested in using computer vision to achieve something usable in real world problems. 

From working part time at a grocery store i noticed how some stores optimize time in store by putting more in demand items in the back forcing customers to see more products. I wanted to see if there was a way to track this using machine learning.


Innovation Paragraph.
Instead of just simply tracking customers using camera vision, we plan to implement the data to get useful information out of it that can have real use benefits. The system will attempt to track individual anonymous customers over time and convert their movement into useful spatial analytics and data.

Problem Statement: Combing through camera footage everyday to get data for businesses is too time consuming for the payoff it gives. Instead we can have machine learning do it.
Research question: How accurately can computer vision anonymously track customer movement and generate retail traffic data?
Secondary question: As we add more people for it to track and increase crowd density, how scalable is it?


Input:
Frame number
Bounding box coordinates
Detection confidence 
Temporary tracking ID 
X position
Y position 
Output:
	The system will output detected and tracked people in the video along with temporary anonymous tracking IDs.

From the data we can derive easily found numbers too like: total customer count, current occupancy, estimated dwell time, traffic by time period, tracking accuracy, and a customer traffic heatmap.

ML Techniques:
	Detection: YOLOv8 (or YOLOv5/v11) for person detection 
Tracking: DeepSORT or ByteTrack for assigning temporary IDs across frames
Downstream analytics: This is a post-processing/aggregation step, not a separate ML model.

Dataset link/source:
https://personal.ie.cuhk.edu.hk/~ccloy/downloads_mall_dataset.html (Mall data set)
https://www.merl.com/research/highlights/merl-shopping-dataset (Inside convenience stores)
https://motchallenge.net/data/MOT17/ (Crowd tracking dataset)

Dataset size: 
Mall data set: Over 2000 Frames/Images, 21 training sets, 21 testing sets of video.
Merl dataset: 106 videos of people shopping in a grocery-store setting, recorded from a fixed overhead camera. Each video is about two minutes long.


Google Doc:https://docs.google.com/document/d/1BGIz4XP0IiK460JMR4NI85d7eCQkjVdntJ2RPkOFDP4/edit?usp=sharing

