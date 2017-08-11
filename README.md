# QRScanner

QRScanner demo application on QT + QZXing




QRScanner QT demo running on i.MX6UL

https://community.nxp.com/docs/DOC-333910

由 Eric Chen 员工 于 2017-2-23 创建的文档版本 1喜欢 • 显示 1 喜欢1 评论 • 4
以全屏模式查看
Overview
As more and more communication required between online and offline, the QR code is widely used in the mobile payment, mobile small apps, industry things identification and etc. The i.MX6UL/ULL has the IP of CSI and PXP for camera connection and image CSC/FLIP/ROTATION acceleration. A LCDIF IP is supporting the display, but no 3D IP support. This means this low power and low end AP is very suitable for the industry HMI segment, which does not require a cool 3D graphic display, but a simple and straightforward GUI for interaction. QR code scanner is one of the use cases in the industry segment, which more and more customer are focusing on.
The i.MX6UL CPU freq of i.MX6UL is about 500Mhz, and it does not have GPU IP, so a lightweight GUI and window system is required. Here we recommend the QT with wayland backend (without X11), which would make the window system small and faster than traditional X11 UI. Why chose QT is because of it has open source version, rich components, platform independent, good performance for embedded system and strong development staffs like QtCreator for creating application.
 
How to enable the QT development environment, check this:
Enable QT developement for i.MX6UL 
 
Here I made a QR code scanner demo based on QT5.6 + QZXing (QR/Bar code scan engine) running on the i.MX6UL EVK board with a UVC camera (at least 640x480 resolution is required) and 480x272px LCD.
 
Source code is open here (License Apache2.0):
https://github.com/muddog/QRScanner 
 
 
Implementation
To do camera preview and capture, you must think on the gstreamer first, which is easy use and has the acceleration pads which implemented by NXP for i.MX6UL. Yes, it's very easy for you to enable the preview in console like:
$ gst-launch-1.0 v4l2src device=/dev/video1 ! video/x-raw,format=YUY2,width=640,height=320 ! imxvideoconvert_pxp ! video/x-raw,format=RGB16 ! waylandsink
It works under the i.MX6UL EVK, with PXP IP to do color space convert from YUY2 -> RGB16 acceleration, also the potential scaling of the image. The CPU loading of this is about 20-30%, but if you use the component of "videoconvert" to replace the "imxvideoconvert_pxp", we do CSC and scale by CPU, then the loading would increase to 50-60%. The "/dev/video1" is the device node for UVC camera, it may different in your environment.
So our target is clear, create such pipeline (with PXP acceleration) in the QT application, and use a appsink to get preview images, do simple "sink" to one QWidget by drawing this image on the widget surface for preview (say every 50ms for 20fps). Then in other thread, we fetch the preview buffer in a fixed frequency (like every 0.5s), then feed it into the ZXing engine to decode the strings inside this image.
 
Here are the class created inside the source code:
ScannerQWidgetSink
It act as a gstreamer sink for preview rendering. Init the pipeline, create a timer with timeout every 50ms. In the timer handler, we use appsink to copy the camera buffer from gstreamer, and tell the ViewfinderWidget to do update (re-draw event).
ViewfinderWidget
This class inherit from the QWidget, which draw the preview buffer as a QImage onto it's own surface by using QPainter. The QImage is created at the very begining with the image buffer created by the ScannerQWidgetSink. Because QImage itself does not maintain the image buffer, so the buffer must be alive during it's usage. So we keep this buffer during the ScannerQWidgetSink life cycle, copy the appsink buffer from pipeline to it for preview.
MainWindow
Create main window, which does not have title bar and border. Start any animation for the red line scan bar. Create instance of DecoderThread and ScannerQWidgetSink. Setup and start them.
DecoderThread
A infinite loop, to wait for a available buffer released by the ScannerQWidgetSink every 0.5s. Copy the buffer data to it's own buffer (imgData) to avoid any change to the buffer by sink when doing decoding. Then feed this copy of buffer into ZXing engine to get decoder result. Then show on the QLabel.
 
Screenshot under wayland (weston) desktop:

 
 
Customize
Camera instance
Now I use the UVC camera which pluged in the USB host, which device node is /dev/video1. If you want to use CSI or other device, please change the construction parameters for ScannerQWidgetSink():
sink = new ScannerQWidgetSink(ui->widget, QString("v4l2src device=/dev/video1"));
Image resolution captured and review
Change the static member value of ScannerQWidgetSink class:
uint ScannerQWidgetSink::CAPTURE_HEIGHT = 480;
uint ScannerQWidgetSink::CAPTURE_WIDTH = 640;
Preview fps and decoding frequency
Find the "framerate=20/1" strings in the ScannerQWidgetSink::GstPipelineInit(), change to your fps. You also have to change the renderTimer start timeout value in the ::StartRender(). The decoding frequency is determined by renderCnt, which determine after how many preview frames showed to feed the decoder.
Main window size
It's fixed size of main window, you have to change the mainwindow.ui. It's easy to do in the QtCreate Designer.
 
FAQ
Why not use CSI camera in demo?
Honestly, I do not have CSI camera module, it's also DNP when you buying the board on NXP.com. So a widely used UVC camera is preferred, it's also easy for you to scan QR code on your phone, your display panel etc.
 
Why not use QCamera to do preview and capture?
The QCamera class in the Qtmultimedia component uses the camerabin2 gstreamer plugin, which create a very long pipeline for different usage of viewfinder, image capture and video encoder. Camerabin2 would eat too much CPU and memory resource, take picture and recording are very very slow. The preview of 30fps would eat about 70-80% CPU loading even I hacked it using imxvideoconvert_pxp instread of software videoconvert. Finally I give up to implement the QRScanner based on QCamera.
 
How to make sure only one instance of QT app is running?
We can use QSharedMemory to create a share memory with a unique KEY. When second instance of app is started, it would check if the share memory with this KEY is created or not. If the shm is there, it means there's already one instance running, it has to exit().
But as the QT mentioned, the QSharedMemory can not be destroyed correctly when app crashed, this means we have to handle each terminate signal, and do delete by ourselves:
static QSharedMemory *gShm = NULL;
static void terminate(int signum)
{
   if (gShm) {
      delete gShm;
      gShm = NULL;
   }
   qDebug() << "Terminate with signal:" << signum;
   exit(128 + signum);
}
int main(int argc, char *argv[])
{
   QApplication a(argc, argv);
   // Handle any further termination signals to ensure the
   // QSharedMemory block is deleted even if the process crashes
   signal(SIGHUP, terminate ); // 1
   signal(SIGINT, terminate ); // 2
   signal(SIGQUIT, terminate ); // 3
   signal(SIGILL, terminate ); // 4
   signal(SIGABRT, terminate ); // 6
   signal(SIGFPE, terminate ); // 8
   signal(SIGBUS, terminate ); // 10
   signal(SIGSEGV, terminate ); // 11
   signal(SIGSYS, terminate ); // 12
   signal(SIGPIPE, terminate ); // 13
   signal(SIGALRM, terminate ); // 14
   signal(SIGTERM, terminate ); // 15
   signal(SIGXCPU, terminate ); // 24
   signal(SIGXFSZ, terminate ); // 25
   gShm = new QSharedMemory("QRScannerNXP");
   if (!gShm->create(4, QSharedMemory::ReadWrite)) {
      delete gShm;
      qDebug() << "Only allow one instance of QRScanner";
      exit(0);
   }
 
.....
