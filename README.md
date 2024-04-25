
# The Best WAV Player

A bluetooth WAV player with ULCD integration - Georgia Tech ECE4180


## Authors

- Vincent Kosasih - [@kosasih21](https://www.github.com/kosasih21)
- George Lee - [@glee366](https://www.github.com/glee366)
- Alejandro Avila - [@aavila33](https://www.github.com/aavila33)


## Demo
View our video project demonstration [**here**](https://youtu.be/IGfztZpOnzM).

Images:
![Reference1](/imgs/IMG_3831.jpg)
![Reference2](/imgs/IMG_3832.jpg)
## Features

- **Audio Playback:** The core functionality of our device is to play WAV audio files, which are typically slightly compressed audio files that provide high-quality sound through a standalone audio playback device.
- **Bluetooth Connectivity:** With the BlueART Bluetooth chip, our WAV player can receive audio commands wirelessly from a Bluetooth-enabled device. This allows for easily expandable wireless controls for the player.
- **SD Card Storage:** The SD card reader enables our device to read WAV files stored on an SD card, providing a portable, local, and expandable storage solution for audio files.
- **Display Interface:** The uLCD (micro Liquid Crystal Display) is used as a customizable display that can show track information and other user interface elements.



## Parts and Schematics
- Mbed LPC1768, [Shop Link](https://www.sparkfun.com/products/retired/14458), [Reference Page](https://os.mbed.com/platforms/mbed-LPC1768/)
- LCD Display uLCD-144G2, [Shop Link](https://www.sparkfun.com/products/11377), [Reference Page](https://os.mbed.com/users/4180_1/notebook/ulcd-144-g2-128-by-128-color-lcd/)
- Adafruit Bluefruit LE UART, [Shop Link](https://www.adafruit.com/product/2479), [Reference Page](https://os.mbed.com/users/4180_1/notebook/adafruit-bluefruit-le-uart-friend---bluetooth-low-/)
- Speaker, [Shop Link](), [Reference Page](https://os.mbed.com/users/4180_1/notebook/using-a-speaker-for-audio-output/)
- NPN General Purpose Amplifier, [Shop Link](https://www.jameco.com/z/PN3565-Major-Brands-PN3565-NPN-General-Purpose-Amplifier-TO-92-25V-0-5A_787608.html), [Reference Page](https://os.mbed.com/users/4180_1/notebook/using-a-speaker-for-audio-output/)
- Potentiometer, [Shop Link](https://www.sparkfun.com/products/9806), [Reference Page](https://os.mbed.com/users/jderiso2/notebook/potentiometer/)
- External Power Supply
- Various Wires
Miscellaneous:
- Phone for bluetooth connectivity
- Breadboard
Schematics and Wiring:
***Insert image here***
## Source Code
*If intended to use, please be sure to utilize the mbed-rtos library*
```cpp
#include "mbed.h"
#include "rtos.h"
#include "uLCD_4DGL.h"
#include "SDFileSystem.h"
#include "wave_player.h"
#include <cstdio>
#include <exception>

// today, implement ulcd

SDFileSystem sd(p5, p6, p7, p8, "sd"); //SD card
uLCD_4DGL uLCD(p9, p10, p11);
AnalogOut DACout(p18);
PwmOut volume(p26);
wave_player waver(&DACout);


DigitalOut led1(LED1);
DigitalOut led2(LED2);
RawSerial blue(p28,p27);
Thread blueT;
Thread wav_thread;
Thread lcd;
Mutex mtx;
FILE *wave_file;


// Assuming a fixed list of songs for demonstration
int currentTrack = 0; // Index of the current song
bool isPlaying = true; // Playback state


const int MAX_FILES = 5; // Max number of audio files
char* playlist[MAX_FILES]; // Array to store filenames
char* songName[MAX_FILES];
int fileCount = 0; // Number of files in the playlist
char* artists[MAX_FILES];
char* songs[MAX_FILES];

void scanDirectory(const char* dirPath) {
    DIR* dir = opendir(dirPath);
    if (dir == NULL) {
        printf("Failed to open directory\n");
        return;
    }

    struct dirent* ent;
    while ((ent = readdir(dir)) != NULL && fileCount < MAX_FILES) {
        // Check if the file is a WAV file
        if (strstr(ent->d_name, ".wav") != NULL) {
            int nameLen = strlen(ent->d_name);
            songName[fileCount] = new char[nameLen + 2];
            strcpy(songName[fileCount], ent->d_name);
            int dirLen = strlen(dirPath);
            playlist[fileCount] = new char[dirLen + nameLen + 2]; // Allocate memory for the filename + path + null terminator
            strcpy(playlist[fileCount], dirPath); // Copy the directory path first
            if (dirPath[dirLen - 1] != '/') {
                strcat(playlist[fileCount], "/"); // Ensure there is a slash between the directory and the filename
            }
            strcat(playlist[fileCount], ent->d_name); // Append the filename
            fileCount++;
        }
    }
    closedir(dir);
}
void blackOut() {
    uLCD.locate(0,14);
    uLCD.color(BLACK);
    uLCD.printf(songName[currentTrack]);
}
void current_song() {
    
    uLCD.filled_rectangle(0, 112, 128, 128, BLACK);
    uLCD.color(RED);
    //uLCD.printf("                                                                            ");
    uLCD.locate(0,14);
    uLCD.printf(artists[currentTrack]);
    uLCD.locate(0,15);
    uLCD.printf(songs[currentTrack]);
}

void play_wav(const char *filename) {
    //mtx.lock();
    wave_file = fopen(filename, "r");
    if (wave_file == NULL) {
        printf("Error opening file: %s\n", filename);
        return;
    }
    //mtx.unlock();
    printf("Playing: %s\n", filename);
    waver.play(wave_file);
    fclose(wave_file);
    //blackOut();
    currentTrack = (currentTrack + 1) % (sizeof(playlist) / sizeof(playlist[0]));
    printf("\n %d \n", currentTrack);
    current_song();
    play_wav(playlist[currentTrack]);

}
 


//create a new play wav when its skipped kill thread, 

//sleep threads with play/pause

void playNextTrack() {
    currentTrack = (currentTrack + 1) % (sizeof(playlist));
    play_wav(playlist[currentTrack]);
    isPlaying = true;
}

void playPreviousTrack() {
    currentTrack = (currentTrack - 1 + sizeof(playlist) / sizeof(playlist[0])) % (sizeof(playlist) / sizeof(playlist[0]));
    play_wav(playlist[currentTrack]);
    isPlaying = true;
}

void togglePlayPause() {
    if (isPlaying) {
        // Assuming we cannot pause, as `wave_player` lacks this feature.
        // For demonstration, this will just stop the current playback.
        isPlaying = false;
        // Ideally, implement pause functionality here if possible.
    } else {
        if (!isPlaying && currentTrack >= 0) {
            play_wav(playlist[currentTrack]);
            isPlaying = true;
        }
    }
}



int main() {
    printf("\r\n\nHello, wave world!\n\r");
    printf("Scanning SD card...\n");
    scanDirectory("/sd/");
    uLCD.text_width(1); //4X size text
    uLCD.text_height(1);
    uLCD.locate(4,1);
    uLCD.printf("WAV Player");
    
    // Now you can use the `playlist` array up to `fileCount` elements
    int yStart = 2; // Start from row 2 right under the title to minimize space

for (int i = 0; i < fileCount; i++) {
    printf("File %d: %s\n", i, songName[i]);

    // Split the song name into artist and title
    char* artist = strtok(songName[i], "-");
    artists[i] = artist;

    char* title = strtok(NULL, "-");
    songs[i] = title;
    // Remove any leading spaces from the title
    if (title && title[0] == ' ') {
        title++;
    }

    // Set the text properties
    uLCD.textbackground_color(BLACK);
    uLCD.text_width(1); // Standard text width
    uLCD.text_height(1); // Standard text height

    // Calculate where to position the text
    int yPosition = yStart + i * 2; // Each song gets 2 lines

    // Display the artist
    uLCD.color(GREEN); // Set artist name to green
    uLCD.locate(0, yPosition);
    uLCD.printf("%d)%s", i+1, artist);

    // Display the song title
    uLCD.color(BLUE); // Set song title to blue
    uLCD.locate(0, yPosition + 1);
    uLCD.printf("%s", title); // Removed indentation
}
    uLCD.locate(3,13);
    uLCD.printf("Now Playing:");
    current_song();
    //freePlaylist();
    Thread::wait(1000);
    wav_thread.start(callback(play_wav, playlist[currentTrack]));
    
    char bnum = 0;
    char bhit = 0;
    while (1) {
        if (blue.getc() == '!') {
            if (blue.getc() == 'B') { // Button data packet
                bnum = blue.getc(); // Button number
                bhit = blue.getc(); // 1=hit, 0=release
                if (blue.getc() == char(~('!' + 'B' + bnum + bhit))) { // Checksum OK?
                    switch (bnum) {
                        case '1': //number button 1
                            if (bhit=='1') {
                                 isPlaying = true;
                                wav_thread.terminate();
                                Thread::wait(500);
                                printf("\n 1st track \n");
                                //fclose(wave_file);
                                //blackOut();
                                currentTrack = 0;
                                printf("\n %d \n", currentTrack);
                                Thread::wait(500);
                                current_song();
                                wav_thread.start(callback(play_wav, playlist[currentTrack]));
                            } 
                            break;
                        case '2': //number button 2
                            if (bhit=='1') {
                                //add hit code here
                                 isPlaying = true;
                                wav_thread.terminate();
                                Thread::wait(500);
                                printf("\n 2nd track \n");
                                //fclose(wave_file);
                                //blackOut();
                                currentTrack = 1;
                                printf("\n %d \n", currentTrack);
                                Thread::wait(500);
                                current_song();
                                wav_thread.start(callback(play_wav, playlist[currentTrack]));
                            }
                            break;
                        case '3': //number button 3
                            if (bhit=='1') {
                                 isPlaying = true;
                                wav_thread.terminate();
                                Thread::wait(500);
                                printf("\n 3rd track \n");
                                //fclose(wave_file);
                                //blackOut();
                                currentTrack = 2;
                                printf("\n %d \n", currentTrack);
                                Thread::wait(500);
                                current_song();
                                wav_thread.start(callback(play_wav, playlist[currentTrack]));
                            } 
                            break;
                        case '4': //number button 4
                            if (bhit=='1') {
                                 isPlaying = true;
                                wav_thread.terminate();
                                Thread::wait(500);
                                printf("\n 4th track \n");
                                //fclose(wave_file);
                                //blackOut();
                                currentTrack = 3;
                                printf("\n %d \n", currentTrack);
                                Thread::wait(500);
                                current_song();
                                wav_thread.start(callback(play_wav, playlist[currentTrack]));
                            }
                            break;
                        case '5': // Button 5 up arrow for play toggle
                            if (bhit == '1') {
                                //togglePlayPause();
                                printf("\n play track \n");
                                Thread::wait(500);
                                wav_thread.start(callback(play_wav, playlist[currentTrack]));
                                isPlaying = true;
                             
                            }
                            break;
                        case '6': // Button 6 down arrow for pause toggle
                            if (bhit == '1') {
                                //togglePlayPause();
                                wav_thread.terminate();
                                Thread::wait(500);
                                printf("\n pause track button \n");
                                isPlaying = false;
                            }
                            break;
                        case '7': // Button 7 left arrow for previous song
                            if (bhit == '1') {
                                isPlaying = true;
                                printf("\n previous track \n");
                                wav_thread.terminate();
                                Thread::wait(500);
                                //fclose(wave_file);
                                //blackOut();
                                currentTrack = (currentTrack - 1) % (sizeof(playlist) / sizeof(playlist[0]));
                                printf("\n %d \n", currentTrack);
                                Thread::wait(500);
                        
                                current_song();
                                wav_thread.start(callback(play_wav, playlist[currentTrack]));
                            }
                            break;
                        case '8': // Button 8 right arrow for next song
                            if (bhit == '1') {
                                isPlaying = true;
                                wav_thread.terminate();
                                Thread::wait(500);
                                printf("\n next track \n");
                                //fclose(wave_file);
                                //blackOut();
                                currentTrack = (currentTrack + 1) % (sizeof(playlist) / sizeof(playlist[0]));
                                printf("\n %d \n", currentTrack);
                                Thread::wait(500);
                                current_song();
                                wav_thread.start(callback(play_wav, playlist[currentTrack]));
                            }
                            break;
                        default:
                            break;
                    }
                }
            }
        }
    }
}


```
