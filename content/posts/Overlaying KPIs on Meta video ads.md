---
title: "Overlaying KPIs on Meta video ads"
date: "2024-11-04"
summary: "Combining the power of Supermetrics with open source libraries to overlay KPIs on video frames"
toc: false
readTime: true
autonumber: false
math: true
---
**The problem**:

Out of the box video metrics in Meta (Video Plays, 3 second video plays, % of 50% Play, % of 100% Play) or custom metrics (scroll stopping rate: 3 second video plays/ video plays) are helpful to analyze video performance.

However, even with access to the data, **it remains challenging for creative teams to contextualize performance**. Doing so requires back and forth between the performance data and videos.

**Solution**:

To make performance insights more actionable to creative teams, **the goal is to overlay performance metrics directly on the video**, making it easy to understand how changes to the ad’s narrative arc impact performance across ad variations.

Specifically, we are looking to draw a play curve of the ad and to overlay the % of users that are still watching the video (retention %) at specific time increments, positioning the metric on the Y axis based on view retention and on the X axis based on video elapsed time.

We’re using the latest iPhone 16 with Apple Intelligence ad and simulated data as an example.

Given [how funny the ad was](https://www.youtube.com/watch?v=3m0MoYKwVTM&t) and how well it was edited for Social Media ads, it's likely it performed well!

![image](/notes/images/AppleAd_Metrics2.jpg)

![image](/notes/images/AppleAd_Metrics3.jpg)

To do this, we need:
- **[Supermetrics](https://www.supermetrics.com)**: to query video performance data from the Meta Ads API
- **[FFmpeg](https://www.ffmpeg.org/)**: to generate still frames from the video
- **[GNUplot](http://www.gnuplot.info/)**: to generate the view rate by frame & the play curve


## Step 1: Retrieving the play curve data with Supermetrics

Using the Supermetrics add-on for Google sheets, we configure the following API query:
- Dimension:  Campaign Name, Ad set Name, Ad Name
- Metrics: CTR, Conversion Rate, average play time, play curve at 0,1,2,3,4,5... seconds

We then set the query to update automatically (once a week or more frequently), update the permissions to "access with the link", publish as CSV and update the google_sheet_url variable with the sheet url directly in the script.

```r
google_sheet_url="[FILE-URL]/pub?gid=0&single=true&output=csv"
```

## Step 2: Setting up a local environment

- Install FFmpeg & GNUplot via **[Homebrew](https://brew.sh/)** in the MacOS Terminal
- Place the script in a local folder named “FFmpeg”
- Place video ads in the folder, following the same naming conventions as the ad unit names used in Meta

## Step 3: Configuring the script

The script is available on **[GitHub](https://github.com/louisbel/tools.git)**  and can handle the following customizations:



- Retrieving data from a Google Sheet, or a manual file
- Setting different font sizes and colors for KPIs
- Customizing the positioning of KPIs
- Generating the summary image with or without the play curve
- Generating an animated gif recap

```r
# Configurable Settings
font_file="${input_folder}/Roboto.ttf"  # Path to the font file
frames_per_row=4                        # Number of frames per row in the final montage
output_width=900                        # Assumed output width for positioning
output_height=1700                      # Assumed output height for positioning
summary_metrics_y_offset_start=1000     # Initial y-offset for summary metrics
font_summary_size=60                    # Font size for summary metrics
summary_font_color="white"              # Font color for text
timestamp_font_size=80                  # Font size for timestamp text
timestamp_font_color="white"            # Font color for timestamp
position_timestamp_x="0.75"             # Timestamp horizontal position as a percentage of width
position_timestamp_y="0.1"              # Timestamp vertical position as a percentage of height
play_metric_font_size=120               # Font size for play metrics
play_metric_font_color="orange"         # Font color for play metrics
background_color="black@0.7"            # Background box color and transparency for all metrics and text
display_play_curve="Y"                  # Display play curve graph (Y/N)
position_playcurve_x=50                 # X position of play curve on first frame
position_playcurve_y=800                # Y position of play curve on first frame
create_gif="Y"                          # Set to "Y" to create a GIF, "N" to skip GIF creation
```

## Step 4: Running the script

Using the Terminal, we navigate to the local folder:
```r
Command: cd Users/MYNAME/FFmpeg
```

Give the script executing permission, and run the script:
```r
Command: chmod +x process_video.sh
Command: ./process_video.sh.
```

The script will output the video KPI overview and the animated gif in the overviews folder.


---

## Future improvement ideas:

- Ranking the variant based on scroll stopping rate vs other variants in the same batch
- Scheduling an automated script run to automatically generate updated overviews
- Have the script move the output overview images to a shared Drive for easier collaboration



