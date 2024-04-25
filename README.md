
# The Best WAV Player

A bluetooth WAV player with ULCD integration - Georgia Tech ECE4180


## Authors

- Vincent Kosasih - [@kosasih21](https://www.github.com/kosasih21)
- George Lee - [@glee366](https://www.github.com/glee366)
- Alejandro Avila - [@aavila33](https://www.github.com/aavila33)


## Demo
View our video project demonstration [**here**](https://www.youtube.com/watch?v=dQw4w9WgXcQ).

Images:
XXX
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
- External Power Supply
- Various Wires
Miscellaneous:
- Phone for bluetooth connectivity
- Breadboard
Schematics and Wiring:
***Insert image here***
## Source Code
```cpp
#include "mbed.h"
#include "rtos.h"
#include "uLCD_4DGL.h"
#include "SDFileSystem.h"
#include "wave_player.h"
#include <cstdio>
#include <exception>


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
Mutex mtx;
FILE *wave_file;

// Assuming a fixed list of songs for demonstration
const char* playlist[] = {"/sd/sample.wav", "/sd/sample1.wav"};
int currentTrack = 0; // Index of the current song
bool isPlaying = true; // Playback state


void play_wav(const char *filename) {
    wave_file = fopen(filename, "r");
    if (wave_file == NULL) {
        printf("Error opening file: %s\n", filename);
        return;
    }
    printf("Playing: %s\n", filename);
    waver.play(wave_file);
    fclose(wave_file);
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
    Thread::wait(1000);
    wav_thread.start(callback(play_wav, playlist[currentTrack]));
    
    char bnum = 0;
    char bhit = 0;
    while (1) {
        if(!isPlaying) {
            printf("\n pausing \n");
            //wav_thread.signal_wait(isPlaying, osWaitForever);
            wav_thread.wait(5000);
        }
        if (blue.getc() == '!') {
            if (blue.getc() == 'B') { // Button data packet
                bnum = blue.getc(); // Button number
                bhit = blue.getc(); // 1=hit, 0=release
                if (blue.getc() == char(~('!' + 'B' + bnum + bhit))) { // Checksum OK?
                    switch (bnum) {
                        case '5': // Button 5 up arrow for play toggle
                            if (bhit == '1') {
                                //togglePlayPause();
                                printf("\n play track \n");
                                isPlaying = true;
                            }
                            break;
                        case '6': // Button 6 down arrow for pause toggle
                            if (bhit == '1') {
                                //togglePlayPause();
                                printf("\n pause track button \n");
                                isPlaying = false;
                            }
                            break;
                        case '7': // Button 7 left arrow for previous song
                            if (bhit == '1') {
                                isPlaying = true;
                                printf("\n previous track \n");
                                wav_thread.terminate();
                                //fclose(wave_file);
                                currentTrack = (currentTrack - 1) % (sizeof(playlist) / sizeof(playlist[0]));
                                printf("\n %d \n", currentTrack);
                                wav_thread.start(callback(play_wav, playlist[currentTrack]));
                            }
                            break;
                        case '8': // Button 8 right arrow for next song
                            if (bhit == '1') {
                                isPlaying = true;
                                wav_thread.terminate();
                                printf("\n next track \n");
                                //fclose(wave_file);
                                currentTrack = (currentTrack + 1) % (sizeof(playlist) / sizeof(playlist[0]));
                                printf("\n %d \n", currentTrack);
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
