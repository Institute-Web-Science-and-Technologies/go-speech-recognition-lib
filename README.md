# go-speech-recognition-lib

This library, written in GoLang, provides functions needed to transcribe speech to text using Google's "Cloud Speech-To-Text" API.
It is designed to be used in a C++ Program and to provide an easy to use C++ Speech-To-Text solution.
	 


The following instructions will help on how to compile and use the go library in a C++ program.

## Prerequisites
	
First you'll need the Go compiler (for now you have to use the 64 Bit version):
```
https://golang.org/dl/
```

To use the API you need to get the Client library ("go get" uses git so make sure you run the command in the git bash or the "git" environment variable has been set):
```
go get -u cloud.google.com/go/speech/apiv1
```
	
To use the "Cloud Speech-To-Text" API you need an API-Key (see [Google How-To](https://cloud.google.com/speech-to-text/docs/quickstart-client-libraries#before-you-begin)).
You need to set the GOOGLE_APPLICATION_CREDENTIALS environment variable to point to your google credentials file (containing your API key):	
cmd:
```
set GOOGLE_APPLICATION_CREDENTIALS=PATH TO\googlecredentials.json
```
PowerShell:
```
$env:GOOGLE_APPLICATION_CREDENTIALS="PATH TO\googlecredentials.json"
```
(support of retrieving the API key directly from the .json might be added in future)

As we are using the cgo compiler (part of the Go compiler installed in the first step) to compile a C++ library  from Go code,
you'll need the [MinGW-w64](https://mingw-w64.org/doku.php/download) compiler (used by cgo).

	
Last but not least: of course you will need a C++ Program to use the library:
You'll need an audiostream represented by short values.
	
	
## Prepare the C++

We will load the library at runtime, so you'll have perform a few more steps than just including a header file.
To achieve this we'll use Windows' HANDLES.
	
First you'll have to add:
```
#include <windows.h>
```
	
Then we declare the type of the function handles we will use (global):
```
typedef int(*INITIALIZE_STREAM)(char* cTranscriptLanguage, int cSampleRate);
typedef int(*SEND_AUDIO)(const short*, int);
typedef char*(*RECEIVE_TRANSCRIPT)();
typedef void(*CLOSE_STREAM)();
typedef char*(*GET_LOG)();
typedef int(*IS_INITIALIZED)();
```
	
Next we want to load the plugin (our .dll). Add this to your function:
```
plugin_handle = LoadLibrary("libgostream.dll");
```
	
With a quick check we ensure that the plugin is loaded properly (else you have to leave the function to avoid runtime errors):
```
if (!plugin_handle)
{
	std::cout << "Could not load plugin." << std::endl;
	return;
}
```
	
When the plugin is loaded we have to load the function handles (it's recommend to declare these variables global to prevent scope problems while using threaded calls):

```
INITIALIZE_STREAM InitializeStream;
SEND_AUDIO SendAudio;
RECEIVE_TRANSCRIPT ReceiveTranscript;
CLOSE_STREAM CloseStream;
GET_LOG GetLog;
IS_INITIALIZED IsInitialized;
```					

```
InitializeStream = reinterpret_cast<INITIALIZE_STREAM>(GetProcAddress(plugin_handle, "InitializeStream"));
SendAudio = reinterpret_cast<SEND_AUDIO>(GetProcAddress(plugin_handle, "SendAudio"));
ReceiveTranscript = reinterpret_cast<RECEIVE_TRANSCRIPT>(GetProcAddress(plugin_handle, "ReceiveTranscript"));
GetLog = reinterpret_cast<GET_LOG>(GetProcAddress(plugin_handle, "GetLog"));
CloseStream = reinterpret_cast<CLOSE_STREAM>(GetProcAddress(plugin_handle, "CloseStream"));
IsInitialized = reinterpret_cast<IS_INITIALIZED>(GetProcAddress(plugin_handle, "IsInitialized"));
```					
	
Ensure that the plugins have been loaded properly (else you have to leave the function to avoid runtime errors):
```	
if (!InitializeStream)
{
	std::cout << "Could not load function InitializeStream." << std::endl;
	return
}
if (!SendAudio)
{
	std::cout << "Could not load function SendAudio." << std::endl;
	return;
}
if (!ReceiveTranscript)
{
	std::cout << "Could not load function ReceiveTranscript." << std::endl;
	return;
}
if (!GetLog)
{
	std::cout << "Could not load function GetLog." << std::endl;
	return;
}
if (!CloseStream)
{
	std::cout << "Could not load function CloseStream." << std::endl;
	return;
}
if (!IsInitialized)
{
	std::cout << "Could not load function IsInitialized." << std::endl;
	return;
}
```
	
And now you are able to call the functions provided by the library.
First initialize the stream (provide a [BCP-47](https://www.rfc-editor.org/rfc/bcp/bcp47.txt) language tag to set the language to be transcribed and the Samplerate of the audio recording as an integer value):
```
InitializeStream("en-US", 16000);
```

	
Then call the send and receive functions (it's recommended to send and receive parallel in seperate threads):
```
int failed = SendAudio(pointerToRecording->data(), pointerToRecording->size());
if (failed != 0) {
	std::string log = GetLog();
	std::cout << "Error:" << log << std::endl;	// Or handle the error like you want to
}
```
Note: size has to be an Integer representing the sample count of the recording.

```
std::string received = ReceiveTranscript();
```


To avoid unwanted behavior check (before calling "SendAudio" or "ReceiveTranscript") if the stream is initialized:
```
int initialized = IsInitialized();	// initialized == 1 if the stream is initialized
if (initialized != 1) {
	// Handle the case, that the stream is not initialized.
}
```


To reverse the initialization process call CloseStream:
```
CloseStream();
```
Note: The implementation of "CloseStream", "SendAudio" and "ReceiveTranscript" is secured by mutex, so you can call "CloseStream" without having to worry about crashes.

	
## Installing the library

Now we are ready to compile the source code to a .dll file.
	
Compile with (run in the same directory as stream.go): 
```
go build -o go-speech-recognition-lib.dll -buildmode=c-shared go-speech-recognition-lib.go
```

In the end you'll have to copy the "libgostream.dll" in the same directory as your compiled C++ program and run your executable.	
		

## Authors

* **Christopher Dreide** - [Drizzy3D](https://github.com/Drizzy3D)

