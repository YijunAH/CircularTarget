# CircularTarget
Detecting circular target via non-linear minimization method

# General Background/Research Purpose

In this study, we explored data collected by automated robots searching for a circular target (around 0.5 meter) in a 15 meter multiply by 8 meter rectangular field which contains other obstacles. With optimization, we believe this method can be generalized into other feature recognition area for imaging process.

![minefield](png/minefield.png?raw=true "minefield")

# Brief Background Information about the datasets

In general, the raw dataset contains log files from 100 different experiments.

The target which the robot was searching for was a circular target with 0.5 m radius.

At each position, the robot recorded time, robot position (x, y) and what it 'sees'. Robot can detects/'sees' objects within 2 meter circle.

In each log file, it includes every location the robot went and the signal collected from nearby envirmoment.

If the robot finds the circular target/or at least think it finds the target, it will stop. If not, the robot will keep searching until experiment time run out (set to be 30 min)

# Data Analysis Method

I divided my study into 7 sections in total.

In the first section, I loaded all the required libraries and 100 log files and briefly looked at their sizes.

In section 2, I defined two functions read_logfile_v2 to clean each log file and convert it into a dataframe and then use PreprocessFileCacheToSQL function to save them into SQL database. In the end, all the data was saved into logs (list with 100 elements).

In section 3, I did some exploratory data analysis with all the log files. The majority of the measurement starts at 0.2 s and observations in every log files are ordered based on time. In most cases, the robot found a circular target/stopped for some reason within 20 min.

When looking at the way of robot moves, we found that robot is moving small distances (less than 0.2 m) at each step. Since in this rectangular searching field, y direction is smaller than x direction, the extreme changes in y are smaller than the ones in x.

We also see bimodal distribution of the velocity (moving speed of robot) may be because of the way which the robot was designed initially. It is moving at either fairy slow speed (less than 0.1 m/s) or much faster speed (0.8 m/s)

In the next section, I visualize robot moving path and also robot last 'look'. The figure below is the moving path of robot made with 100 log files.

Looking at 100 robot moving trace plots below, we found out that the robot always started from bottom left corner indicating with a green dot. The time it takes to find the circular target corresponds to the shift in colors from green to red. The final location where the circular target was found/the robot stopped is marked with a blue x.

![RobotMovingPath](png/RobotMovingPath.png?raw=true "RobotMovingPath")

We defined CombineAllLastLook function to aggregate the last look from 100 log files into a single dataframe called 'LastLook'. We randomly draw 9 out of 100 look from the LastLook and plotted them below. The red circle is the 2 meter range which the robot can detect. And the black line describe what the robot acutally saw from the signal collected. Just looking at these 9 plots, we saw that around 50% of the time the robot did not find the circular target at the end of the measurement.

![LastLookPlot](png/LastLookPlot.png?raw=true "LastLookPlot")

Since robot can only 'see' things with 2 meter circle range, in the following section, we want to find out segments where robot 'see' something which might be interesting and then judge if it is the circular target we are interested. We defined three main functions (getSegments, getWrappedSegments_YG, seperateSegments_YG) to do this.
```{r}
getSegments=
  function(range, threshold = 2){
    # range: a list with 360 in length
    rl = rle(range < threshold)
    
    cursor = 1
    ans = list()
    
    for(i in seq(along = rl$lengths)) {
      if(!rl$values[i]) {
        cursor = cursor + rl$lengths[i]
        next
      }
      ans[[length(ans) + 1]] = seq(cursor, length = rl$lengths[i])
      cursor = cursor + rl$lengths[i]
    }
    ans}

getWrappedSegments_YG =
  function(range, threshold = 2){
    # the same input as getSegments
    segments = getSegments(range, threshold)
    if(length(segments) >= 2) {
      s_first = segments[[1]]
      s_last = segments[[length(segments)]]
      if(s_first[1] == 1 && s_last[length(s_last)] == 360) {
        segments[[1]] = c(s_last, s_first)
        segments = segments[-length(segments)]
      }
    }
    segments}

seperateSegments_YG<-function(idx, x, y, threshold=0.15){
  # seperate segments if the distance between the two points exceeds 0.15 threshold
  xdiff<-diff(x[idx])
  ydiff<-diff(y[idx])
  distance<-sqrt((xdiff)^2+(ydiff)^2)
  if (any(distance>threshold)){
    i = which(distance > threshold)[1]
    list(idx[1:(i-1)],idx[(i+1):length(idx)])
  }
  else {list(idx)}
}
```

After taking out potential segments, we need to develop a method which can judge if this segment correspond to a 0.5 meter circular target. Function circle.fit takes three inputs: p (a list which contains initial guess of the radius, x and y position of the circular target), x (a list which contains x axis information from what the robot 'sees'), y (a list which contains y axis information from what the robot 'sees'). It summarise the sum of squares between the two radii.

$$\sqrt{((x_i-x_0)^2+(y_i-y_0)^2))-r)}$$
similar to fitting a regression line


In section 7, I define function robotEva_YG() to process each row in LastLook. This function loops over 100 last look from logfiles, extracts the segments and use circle.fit() and nlm() function to determine which segment apprears to be part of the target circle. For each segment, we also evaluate three additional criteria that actually determine whether the segment seems to be the circular target we were expected. 

Firstly, the length of the segment should be greater than 3 (current default). If there is too little points present in one segment, it is not accurate to judge if it is the circular target or some other obstacles. Secondly, as everything else in real world, the radii of the circular target we were searching for is not 0.5 meter in sharp. We set a range (0.475 to 0.7 meters) to this. If it is outside of the range, we assume it is something else. Lastly, we use the final value of the sum of squares function that we are trying to minimize to measure the goodness of fit. We compare the goodness of fit to the threshold with
(result$minimum/length(segment))>error threshold (0.01)

With the results saved in CircleDetection, we know that out of the 100 last look we took from log file, 37 of them encountered circurlar target in the end. We randomly selected 9 out of the 37 looks and plotted below.

![plotswithCircle](png/plotswithCircle.png?raw=true "plotswithCircle")

# Conclusions

In this study, I looked at 100 log files created by robot with the aim of finding a circular target in the end. I did some exploratory data analysis with 100 log files to study how the data was organized and how the robot was moving and under what speed. I used the data to develop a statistical classifier for idnetifying target with non linear minimization and gained an understanding of how well it works. This method could be used in real-time which the robot is moving through the course to detect circular target. It could also be useful to develop robot search strategies that combine information from series of detections and tell the robot where to go next.

# References
This example is taken from: Chapter 4 Processing Robot and Sensor Log Files: Seeking a Circular Target from the book: Case Studies in Data Science with R by Deborah Nolan, University of California, Berkeley and Duncan Temple Lang, University of California, Davis. 

The complete log files can be downloaded from: http://rdatasciencecases.org/Data.html

A typical robot moving trace and experiment enviroment was shown in this figure:
http://rdatasciencecases.org/Data/minefield.png
