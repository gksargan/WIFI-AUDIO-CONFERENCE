# WIFI-AUDIO-CONFERENCE
this code is basically in an application format. we created an application using android studio to communicate through wifi network. it works in the absence of mobile network so that's the advantage of this application
import android.content.Intent;
import android.media.AudioFormat;
import android.media.AudioManager;
import android.media.AudioRecord;
import android.media.AudioTrack;
import android.media.MediaRecorder;
import android.net.wifi.WifiManager;
import android.os.Handler;
import android.os.Message;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.ContextMenu;
import android.view.MenuItem;
import android.view.View;
import android.widget.AdapterView;
import android.widget.ListView;
import android.widget.TextView;
import android.widget.ToggleButton;

import java.net.DatagramPacket;
import java.net.InetAddress;
import java.net.MulticastSocket;
import java.net.NetworkInterface;
import java.util.ArrayList;
import java.util.Enumeration;

import com.example.wificall.list.*;

public class wifiCall extends AppCompatActivity {
    String[] myDetails;
    ArrayList<IpList> ipLists;
    MulticastSocket sendInfoPacket;
    MulticastSocket sendVoicePacket;
    DatagramPacket sendDatagramPacket;
    DatagramPacket receiveDatagramPacket;
    InetAddress groupAddress;
    WifiManager wifiManager;
    WifiManager.MulticastLock multicastLock;
    String ipPacket;
    ListView ipMembersList;
    ArrayList<String> ipArrayList;
    Handler handler;
    int minbuff;

    private final int MY_IP=1;
    private final int NOT_MY_IP=0;
    private final int NULL_IP=2;

    AudioManager audioManager;

    int my_ip=NULL_IP;

    boolean recording=true;
    boolean speakers=true;

    Thread startRecord, startTrack, socketListen;

    private String leaveIPPacket;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_wifi_call);
        findViewById(R.id.speakToggle).setEnabled(false);
        findViewById(R.id.leaveButton).setEnabled(false);
        findViewById(R.id.speakToggle).setEnabled(false);
        audioManager=(AudioManager) getSystemService(AUDIO_SERVICE);
//        Log.d("CSR","is speaker phone on? - "+audioManager.isSpeakerphoneOn());
        //audioManager.setSpeakerphoneOn(false);
        audioManager.setSpeakerphoneOn(true);
//        Log.d("CSR","is speaker phone on? - "+audioManager.isSpeakerphoneOn());
//        Log.d("CSR","is speaker phone on? - "+AudioRecord.getMinBufferSize(44100,AudioFormat.CHANNEL_IN_MONO,AudioFormat.ENCODING_PCM_16BIT));
//        Log.d("CSR","is speaker phone on? - "+AudioTrack.getMinBufferSize(44100,AudioFormat.CHANNEL_OUT_MONO,AudioFormat.ENCODING_PCM_16BIT));
        try {
            //minbuff=AudioRecord.getMinBufferSize(44100, AudioFormat.CHANNEL_IN_MONO,AudioFormat.ENCODING_PCM_16BIT)*10;
            //minbuff=6400;
            minbuff=6400;
            //audioRecord=new AudioRecord(MediaRecorder.AudioSource.MIC,44100,AudioFormat.CHANNEL_IN_MONO,AudioFormat.ENCODING_PCM_16BIT,minbuff);
            //audioRecord=new AudioRecord(MediaRecorder.AudioSource.VOICE_RECOGNITION,8000,AudioFormat.CHANNEL_IN_MONO,AudioFormat.ENCODING_PCM_16BIT,minbuff);
            ipLists=new ArrayList<IpList>();
            ipArrayList=new ArrayList<String>();
            ipMembersList=findViewById(R.id.ipMemberList);
            sendInfoPacket=new MulticastSocket(9999);
            sendVoicePacket=new MulticastSocket(9998);
            groupAddress=InetAddress.getByName("225.4.5.6");
            wifiManager=(WifiManager)getApplicationContext().getSystemService(WIFI_SERVICE);
            multicastLock= wifiManager.createMulticastLock("CSR");
            multicastLock.setReferenceCounted(true);
            multicastLock.acquire();
            Intent intent = getIntent();
//            Log.d("CSR","Entered main screen");
//            Log.d("CSR",intent.toString());
            myDetails = intent.getStringArrayExtra("MYDETAILS");
            Log.d("CSR", myDetails[0] + " - " + myDetails[1] + " - " + myDetails[2]+" - "+myDetails[3]);
            ((TextView)findViewById(R.id.myName)).setText("My Name: "+myDetails[0]);
            ((TextView)findViewById(R.id.myIpText)).setText("My Ip Address: "+myDetails[1]);
            ((TextView)findViewById(R.id.serverIpText)).setText("Server Ip Address: "+myDetails[2]);
            ipPacket="IP:"+myDetails[1]+":"+myDetails[0];

            registerForContextMenu(ipMembersList);

            ipMembersList.setOnItemLongClickListener(new AdapterView.OnItemLongClickListener() {
                @Override
                public boolean onItemLongClick(AdapterView<?> parent, View view, int position, long id) {
                    leaveIPPacket="LE:"+ipLists.get(position).getIpAddress();
                    return false;
                }
            });

            if(myDetails[3].equals("1")){
                try {
//                    Log.d("CSR","Network interface list");
                    NetworkInterface networkInterface;
                    Enumeration<NetworkInterface> n=NetworkInterface.getNetworkInterfaces();

//                    Log.d("CSR","Network interface enumeration- "+n);

                    while (n.hasMoreElements())
                    {
                        if((networkInterface=n.nextElement()).getDisplayName().equals("wlan0"))
                        {
                            Log.d("CSR",networkInterface.getDisplayName()+" CSR - network interface");
                            sendInfoPacket.setNetworkInterface(networkInterface);
                            sendVoicePacket.setNetworkInterface(networkInterface);
                        }
                    }
                } catch (Exception e) {
                    Log.d("CSR","Network interface "+e.toString());
                }
            }

            handler=new Handler(new Handler.Callback() {
                @Override
                public boolean handleMessage(Message msg) {
                    if(msg.what==10){
//                        Log.d("CSR","Inside handler log");
                        Bundle bundle=msg.getData();
                        String strIPPacket=bundle.getString("strIPPacket");
//                        Log.d("CSR","strIPPacket - "+strIPPacket);
                        updateGroupMembers(strIPPacket);
                    }
                    else if(msg.what==11){
//                        Log.d("CSR","Handler log clear ip");
                        Bundle bundle=msg.getData();
                        String strIPPacket=bundle.getString("strIPPacket");
//                        Log.d("CSR","strIPPacket - "+strIPPacket);
                        updateGroupMembers(strIPPacket);
                    }
                    else if(msg.what==12){
//                        Log.d("CSR","Toggle disable");
                        findViewById(R.id.speakToggle).setEnabled(false);
                        findViewById(R.id.leaveButton).setEnabled(false);
                    }
                    else if(msg.what==13){
//                        Log.d("CSR","Toggle enable");
                        findViewById(R.id.speakToggle).setEnabled(true);
                        findViewById(R.id.leaveButton).setEnabled(true);
                        ((TextView)findViewById(R.id.groupMembersText)).setText("Group Members");
                    }
                    else if(msg.what==14){
//                        Log.d("CSR","Speaker notify");
                        Bundle bundle=msg.getData();
                        String strIPPacket=bundle.getString("strIPPacket");
//                        Log.d("CSR","strIPPacket - "+strIPPacket);
                        updateGroupMembers(strIPPacket);
                    }
                    return false;
                }
            });

            socketListenStart();
//            Log.d("CSR","Current thread name - "+Thread.currentThread().getName());

            findViewById(R.id.joinButton).setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    try {
                        //socketListenStart();
//                        Log.d("CSR","Passed here");
                        sendInfoPacket.joinGroup(groupAddress);
                        sendVoicePacket.joinGroup(groupAddress);
                        Log.d("CSR","Joined the group");
                        broadcastIP();
                        findViewById(R.id.joinButton).setEnabled(false);
                        findViewById(R.id.speakToggle).setEnabled(true);
                        findViewById(R.id.leaveButton).setEnabled(true);
                    }catch (Exception e){
                        Log.d("CSR","Join - "+e.toString());
                    }
                }
            });

            findViewById(R.id.speakToggle).setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    if(((ToggleButton)(findViewById(R.id.speakToggle))).getText().toString().equals(
                            ((ToggleButton)(findViewById(R.id.speakToggle))).getTextOn().toString()
                    )){
//                        Log.d("CSR","Speak now");
                        new Thread(new Runnable() {
                            @Override
                            public void run() {
                                try {
                                    String infoPacket="SS:"+myDetails[1];
                                    DatagramPacket leavePacket = new DatagramPacket(infoPacket.getBytes(), infoPacket.length(), groupAddress, 9999);
                                    sendInfoPacket.send(leavePacket);
                                }catch (Exception e1){
                                    Log.d("CSR","send info - "+e1.toString());
                                }
                            }
                        }).start();
                        sendData();
                    }
                    else{
//                        Log.d("CSR","Stop speaking");
                        ((TextView)findViewById(R.id.groupMembersText)).setText("Group Members");
                        recording=false;
                        speakers=false;
                        my_ip=NULL_IP;
                        enableToggle();
                        stopSpeakers();
                    }
                }
            });

            findViewById(R.id.leaveButton).setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    try{
                        final String leaveStr="LE:"+myDetails[1];
//                        Log.d("CSR","Leave button clicked");
                        new Thread(new Runnable() {
                            @Override
                            public void run() {
                                try {
                                    DatagramPacket leavePacket=new DatagramPacket(leaveStr.getBytes(),leaveStr.length(),groupAddress,9999);
                                    sendInfoPacket.send(leavePacket);
                                    //sendInfoPacket.leaveGroup(groupAddress);
                                    //sendVoicePacket.leaveGroup(groupAddress);
                                }catch (Exception e1){
                                    Log.d("CSR","Thread leave group - "+e1.toString());
                                }
                            }
                        }).start();
                        findViewById(R.id.joinButton).setEnabled(true);
                        findViewById(R.id.leaveButton).setEnabled(false);
                        findViewById(R.id.speakToggle).setEnabled(false);
                    }catch (Exception e){
                        Log.d("CSR","Leave group - "+e.toString());
                    }
                }
            });

            findViewById(R.id.speakerToggle).setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    if(((ToggleButton)(findViewById(R.id.speakerToggle))).getText().toString().equals(
                            ((ToggleButton)(findViewById(R.id.speakerToggle))).getTextOn().toString()
                    )){
                        audioManager.setSpeakerphoneOn(false);
                    }
                    else{
                        audioManager.setSpeakerphoneOn(true);
                    }
                }
            });

        } catch (Exception e){
            Log.d("CSR","mydet "+e.toString());
        }
    }

    @Override
    public void onCreateContextMenu(ContextMenu menu, View v, ContextMenu.ContextMenuInfo menuInfo) {
        super.onCreateContextMenu(menu, v, menuInfo);
        if(myDetails[0].equals("admin") && myDetails[3].equals(String.valueOf(1)))
            getMenuInflater().inflate(R.menu.leave_menu,menu);
    }

    @Override
    public boolean onContextItemSelected(MenuItem item) {
//        Log.d("CSR","Contect menu leave - "+leaveIPPacket);
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    DatagramPacket leavePacket=new DatagramPacket(leaveIPPacket.getBytes(),leaveIPPacket.length(),groupAddress,9999);
                    sendInfoPacket.send(leavePacket);
                    //sendInfoPacket.leaveGroup(groupAddress);
                    //sendVoicePacket.leaveGroup(groupAddress);
                }catch (Exception e1){
                    Log.d("CSR","Thread leave group - "+e1.toString());
                }
            }
        }).start();
        return super.onContextItemSelected(item);
    }

    @Override
    protected void onDestroy() {
        try{
            Log.d("CSR","OnDestroy invoked");
            final String leaveStr="LE:"+myDetails[1];
            if(startRecord!=null){
                Log.d("CSR","Start record not null - "+startRecord.isAlive());
                if(startRecord.isAlive()==true){
                    startRecord.interrupt();
                    startRecord=null;
                }
            }
            if(startTrack!=null){
                Log.d("CSR","Start track not null - "+startTrack.isAlive());
                if(startTrack.isAlive()==true){
                    startTrack.interrupt();
                    startTrack=null;
                }
            }
            Thread thr=new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        DatagramPacket leavePacket=new DatagramPacket(leaveStr.getBytes(),leaveStr.length(),groupAddress,9999);
                        sendInfoPacket.send(leavePacket);
                        Thread.sleep(20);
                        sendInfoPacket.leaveGroup(groupAddress);
                        sendVoicePacket.leaveGroup(groupAddress);
                        Log.d("CSR","Leave group in onDestroy");
                    }catch (Exception e1){
                        Log.d("CSR","Thread leave group - "+e1.toString());
                    }
                }
            });
            thr.start();
            if(thr!=null){
                Log.d("CSR","Thr not null - "+thr.isAlive());
                while(thr.isAlive()==true){

                }
            }
            Log.d("CSR",socketListen.toString());
            if(socketListen!=null){
                Log.d("CSR","Socket Listen not null");
                if(socketListen.isAlive()){
                    Log.d("CSR","Socket Listen alive");
                    socketListen.interrupt();
                    socketListen=null;
                    Log.d("CSR","Socket Listen null");
                }
            }
            multicastLock.release();
            Log.d("CSR","Finished onDestroy - "+thr.isAlive());
            super.onDestroy();
        }catch (Exception e){
            Log.d("CSR","Leave group - "+e.toString());
        }
    }

    private void stopSpeakers() {
        try{
            final String stopSpea="TP:";
//            Log.d("CSR","Stop speakers");
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        DatagramPacket leavePacket=new DatagramPacket(stopSpea.getBytes(),stopSpea.length(),groupAddress,9999);
                        sendInfoPacket.send(leavePacket);
                    }catch (Exception e1){
                        Log.d("CSR","Thread leave group - "+e1.toString());
                    }
                }
            }).start();
        }catch (Exception e){
            Log.d("CSR","Leave group - "+e.toString());
        }
    }

    private void enableToggle() {
        try{
            final String enableTogg="ET:";
//            Log.d("CSR","Enable toggle");
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        DatagramPacket leavePacket=new DatagramPacket(enableTogg.getBytes(),enableTogg.length(),groupAddress,9999);
                        sendInfoPacket.send(leavePacket);
                    }catch (Exception e1){
                        Log.d("CSR","Thread leave group - "+e1.toString());
                    }
                }
            }).start();
        }catch (Exception e){
            Log.d("CSR","Leave group - "+e.toString());
        }
    }

    private void receiveData() {
        startTrack=new Thread(new receiveVoiceClass());
        startTrack.start();
    }

    private void socketListenStart() {
        socketListen=new Thread(new receivePacketsClass());
        socketListen.start();
    }

    private void sendData() {
        recording=true;

        startRecord=new Thread(new Runnable() {
            @Override
            public void run() {
                AudioRecord audioRecord=null;
                try{
//                    Log.d("CSR","Rec start");
                    //byte[] audiobuff=new byte[6400];
                    audioRecord=new AudioRecord(MediaRecorder.AudioSource.VOICE_RECOGNITION,8000,AudioFormat.CHANNEL_IN_MONO,AudioFormat.ENCODING_PCM_16BIT,minbuff);
                    audioRecord.startRecording();

                    while(recording==true){
                        if(my_ip==MY_IP) {
                            byte[] audiobuff=new byte[6400];
//                            Log.d("CSR","RECORDING STARTS+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++");
                            int byte_a_read = audioRecord.read(audiobuff, 0, audiobuff.length);
//                            Log.d("CSR", "byte-a-read - " + byte_a_read);
                            DatagramPacket datagramPacket = new DatagramPacket(audiobuff, byte_a_read, groupAddress, 9998);
                            sendVoicePacket.send(datagramPacket);
//                            Log.d("CSR", "byte-a-read - "+byte_a_read+" Voice sent");
                            Thread.sleep(20);
                        }
                    }
//                    Log.d("CSR","RECORDING ENDS");
                    audioRecord.stop();
                    audioRecord.release();
                }catch (Exception e){
                    Log.d("CSR","send voice - "+e.toString());
                    if(e.toString().equals("java.lang.InterruptedException")){
                        Log.d("CSR","RECORDING ENDS in interrupt");
                        audioRecord.stop();
                        audioRecord.release();
                    }
                }
            }
        });
        startRecord.start();
        if(startRecord.isInterrupted()){
            Log.d("CSR","Record thread interrupted");
        }
    }

    private class receiveVoiceClass extends Thread{

        AudioTrack audioTrack;
        @Override
        public void run() {
            super.run();
            //byte[] receivedData=new byte[6400];
            speakers=true;
            audioTrack = new AudioTrack(AudioManager.STREAM_VOICE_CALL, 8000, AudioFormat.CHANNEL_OUT_MONO,
                    AudioFormat.ENCODING_PCM_16BIT, 6400, AudioTrack.MODE_STREAM);
            audioTrack.play();

            while (speakers){
                if(my_ip==NOT_MY_IP) {
                    try {
                        audioTrack.flush();
                        byte[] receivedData=new byte[6400];
//                        Log.d("CSR","SPEAKER STARTS");
                        DatagramPacket receiveVoicePacket = new DatagramPacket(receivedData, receivedData.length);
//                        Log.d("CSR", "Voice Socket listening...");
                        sendVoicePacket.receive(receiveVoicePacket);
                        if(!receiveVoicePacket.getAddress().toString().equals("/"+myDetails[1])) {
//                            Log.i("CSR", "Packet received: " + receiveVoicePacket.getLength() + "----" + receiveVoicePacket.getAddress().toString());
                            audioTrack.write(receiveVoicePacket.getData(), 0, 6400);
                        }
                    } catch (Exception e) {
                        Log.d("CSR", "Multisocket receive " + e.toString());
                    }
                }
            }
            audioTrack.stop();
            audioTrack.release();
            Log.d("CSR","Audio stopped and released - run thread");
        }

        @Override
        public void interrupt() {
            super.interrupt();
            try {
                audioTrack.stop();
                audioTrack.release();
                Log.d("CSR", "Audio stopped and released");
            }catch (Exception e2){
                Log.d("CSR","Audio track thread - "+e2.toString());
            }
        }
    }

    private void broadcastIP() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try{
                    //Log.d("CSR","Current thread name - "+Thread.currentThread().getName());
                    Thread.sleep(30);
//                    Log.d("CSR","Broadcastip starts");
                    sendDatagramPacket=new DatagramPacket(ipPacket.getBytes(),ipPacket.length(),groupAddress,9999);
                    sendInfoPacket.send(sendDatagramPacket);
                    Log.d("CSR","IP Packet "+ipPacket+" sent");
                } catch (Exception e){
                    Log.d("CSR","Multicast send socket "+e.toString());
                }
            }
        }).start();
    }

    private class receivePacketsClass extends Thread{
        @Override
        public void run() {
            super.run();
            while (true){
                try {
                    byte[] receivedData=new byte[1000];
                    receiveDatagramPacket = new DatagramPacket(receivedData,receivedData.length);
                    Log.d("CSR","Socket listening...");
                    sendInfoPacket.receive(receiveDatagramPacket);
                    String str_raw = new String(receivedData,0,receiveDatagramPacket.getLength());
                    String str=str_raw.trim();
                    Log.d("CSR","Trimmed IP Packet: "+str);
                    if(str.substring(0,3).equals("IP:")){
                        String ip=str.substring(3,str.indexOf(":",3));
                        Log.d("CSR","ip = "+ip);
                        if(!ipArrayList.contains(ip)){
                            Log.d("CSR",ip+"  not available...hence adding.....................................");
                            ipArrayList.add(ip);
                            //broadcastIP();
                            Message message=Message.obtain();
                            message.what=10;
                            Bundle bundle=new Bundle();
                            bundle.putString("strIPPacket",str);
                            message.setData(bundle);
                            handler.sendMessage(message);
                        }
                    }
                    else if(str.substring(0,3).equals("LE:")){
                        String ip=str.substring(3);
                        Log.d("CSR","ip = "+ip);
                        if(ip.equals(myDetails[1])){
                            sendInfoPacket.leaveGroup(groupAddress);
                            sendVoicePacket.leaveGroup(groupAddress);
                            Log.d("CSR","Leave group in receivePacketsClass");
                            ipArrayList.clear();
                            Log.d("CSR","Arraylist size - "+ipArrayList.size());
                            Message message=Message.obtain();
                            message.what=11;
                            Bundle bundle=new Bundle();
                            bundle.putString("strIPPacket","clear_self");
                            message.setData(bundle);
                            handler.sendMessage(message);
                        }
                        else{
                            Log.d("CSR","Other's IP");
                            ipArrayList.remove(ip);
                            Message message=Message.obtain();
                            message.what=11;
                            Bundle bundle=new Bundle();
                            bundle.putString("strIPPacket",str);
                            message.setData(bundle);
                            handler.sendMessage(message);
                        }
                    }
                    else if(str.substring(0,3).equals("SS:")){
                        String ip=str.substring(3);
                        Log.d("CSR","ip = "+ip);
                        if(ip.equals(myDetails[1])){
                            my_ip=MY_IP;
                            audioManager.setMicrophoneMute(false);
                        }
                        else{
                            my_ip=NOT_MY_IP;
                            audioManager.setMicrophoneMute(true);
                            Message message=Message.obtain();
                            message.what=12;
                            handler.sendMessage(message);
                        }
                        Log.d("CSR","MY_IP - "+my_ip);
                        receiveData();
                        Message message=Message.obtain();
                        message.what=14;
                        Bundle bundle=new Bundle();
                        bundle.putString("strIPPacket",str);
                        message.setData(bundle);
                        handler.sendMessage(message);
                    }
                    else if(str.equals("ET:")){
                        Message message=Message.obtain();
                        message.what=13;
                        handler.sendMessage(message);
                    }
                    else if(str.equals("TP:")){
                        speakers=false;
                        recording=false;
                        if(startRecord!=null) {
//                            Log.d("CSR","startRecord - not null");
                            if (startRecord.isAlive()) {
                                Log.d("CSR","startRecord - alive");
                                startRecord.interrupt();
                                startRecord = null;
                                Log.d("CSR","startRecord - null");
                            }
                        }
                        if(startTrack!=null) {
//                            Log.d("CSR","startTrack - not null");
                            if (startTrack.isAlive()) {
                                Log.d("CSR","startTrack - alive");
                                startTrack.interrupt();
                                startTrack = null;
                                Log.d("CSR","startTrack - null");
                            }
                        }
                    }
                }catch (Exception e){
                    Log.d("CSR","Multisocket receive "+e.toString());
                }
            }
        }

        @Override
        public void interrupt() {
            super.interrupt();
            Log.d("CSR","Socket Listen interrupted");
        }
    }

    private void updateGroupMembers(String strIPPacket) {
        try {
            if (!(strIPPacket.equals("clear_self"))) {
                String groupMode = strIPPacket.substring(0, 3);
                int index = 0;
                if (groupMode.equals("IP:")) {
                    String ip = strIPPacket.substring(3, strIPPacket.indexOf(":", 3));
                    String name = strIPPacket.substring(strIPPacket.indexOf(":", 3) + 1);
                    ipLists.add(new IpList(ip, name));
                    Log.d("CSR", "Added ip addr " + ip + " and name " + name + "=============================");
                    broadcastIP();
                }
                else if (groupMode.equals("LE:")) {
                    String ip = strIPPacket.substring(3);
                    int i = 0;
                    for (IpList ipList : ipLists
                    ) {
                        if (ipList.getIpAddress().equals(ip))
                            index = i;
                        i = i + 1;
                    }
                    ipLists.remove(index);
                    Log.d("CSR", "Deleted ip addr " + ip + "=============================");
                    if(ip.equals(myDetails[1])) {
                        Log.d("CSR","ip delete - "+ip);
                        findViewById(R.id.joinButton).setEnabled(true);
                        findViewById(R.id.leaveButton).setEnabled(false);
                        findViewById(R.id.speakToggle).setEnabled(false);
                    }
                }
                else if (groupMode.equals("SS:")) {
                    String ip = strIPPacket.substring(3);
                    String speakerName=null;
                    for (IpList ipList : ipLists
                    ) {
                        if (ipList.getIpAddress().equals(ip)){
                            speakerName=ipList.getUserName();
                        }
                    }
                    ((TextView)findViewById(R.id.groupMembersText)).setText("Group Members   ["+speakerName+"  speaking...]");
                    Log.d("CSR", "Group Members   "+speakerName+"  speaking...");
                }
                IpAdapter ipAdapter = new IpAdapter(wifiCall.this, 0, ipLists);
                ipMembersList.setAdapter(ipAdapter);
            } else if (strIPPacket.equals("clear_self")) {
                ipLists.clear();
                Log.d("CSR", "Self clear ip=============================");
                IpAdapter ipAdapter = new IpAdapter(wifiCall.this, 0, ipLists);
                ipMembersList.setAdapter(ipAdapter);
                findViewById(R.id.joinButton).setEnabled(true);
                findViewById(R.id.leaveButton).setEnabled(false);
                findViewById(R.id.speakToggle).setEnabled(false);
            }
            Log.d("CSR","Array List Size - "+String.valueOf(ipLists.size())+" %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%");
        }catch (Exception e){
            Log.d("CSR","Update group - "+e.toString());
        }
    }
}
