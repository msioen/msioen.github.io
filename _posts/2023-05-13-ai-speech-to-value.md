---
layout: post
title: Playing around with speech to text
subtitle: Azure AI Experiments
share-img: https://michielsioen.be/img/social-share-ai-speech.png
---

This year I'm experimenting a lot with the Azure AI offerings. In this post I'll describe some of my experiments with natural language and speech-to-text functionalities.

I purposefully wanted to play around with functionality as close to real-time as possible. A lot of existing and useful AI tooling works asynchronously. You give it a request or a prompt and a bit later you'll get your result. The more context you can provide, the better your result is likely going to be.

I wanted to try out how easy it would be to get useful value basically immediately.

# Exploring the Azure AI offering

Azure has a broad offering related to speech and text functionality in their cognitive services. As this space is rapidly evolving, I'm not going to list too many things here aside from my main entry points into the knowledge base.

- [Cognitive Speech Services](https://azure.microsoft.com/en-gb/products/cognitive-services/speech-services/)
- [Cognitive Service for Language](https://azure.microsoft.com/en-gb/products/cognitive-services/language-service/)

Both cognitive services have a small studio application available which you definitely check out. It's a very cool and easy way to try out some of the functionality on offer. They showcase the main functionalities and some realistic scenarios.

- [Language Studio](https://language.cognitive.azure.com): showcases features like PII extraction, extracting named entities and text summarization.
- [Speech Studio](https://speech.microsoft.com/portal): showcases features like speech-to-text, text-to-speech, pronunciation assessment.

These studios are a great entry point for any use case. You're able to validate the main features, but there are also many direct links to documentation. Every feature also has quick start code and examples to get your own implementation started.

On Microsoft Learn there are also several scenario deep-dives worked out. These are very interesting to read through and see how they put several functionalities together. There doesn't seem to be a direct link available to all deep-dives but you can easily click through from the [main documentation page](https://learn.microsoft.com/en-gb/azure/cognitive-services/speech-service/).

# Getting text input

Getting started with the SDK is straightforward. There are also many examples available in the studio dashboards, official documentation and GitHub spaces. Below you can find the starting code which is on its own enough to provide speech-to-text.

```csharp
  // setup the configuration and recognizer
  var speechConfig = SpeechConfig.FromSubscription(subscriptionKey, serviceRegion);
  speechConfig.SpeechRecognitionLanguage = "nl-BE";

  using var audioConfig = AudioConfig.FromDefaultMicrophoneInput();
  using var recognizer = new SpeechRecognizer(speechConfig, audioConfig);

  // subscribe to relevant events
  recognizer.Recognizing += (s, e) =>
  {
      Console.WriteLine($"Recognizing: {e.Result.Text} @{e.Result.OffsetInTicks} - {e.Result.Duration}");
  };

  recognizer.Recognized += (s, e) =>
  {
      Console.WriteLine($"Recognized: {e.Result.Text} @{e.Result.OffsetInTicks} - {e.Result.Duration}");
  };
  
  // start listening and recognizing text
  await recognizer.StartContinuousRecognitionAsync().ConfigureAwait(false);
```

For my initial implementation I hadn't seen yet that there's an option available to let Azure automatically detect the language. This is great for my use cases as I do tend to come across several languages daily.

There are only some minor changes necessary to provide additional configuration. When this is done, every 'Recognizing' and 'Recognized' event can now indicate the language Azure has identified.

```csharp
  // setup languages to detect
  var languages = new string[] { "nl-NL", "fr-FR", "en-US" };
  var autoDetectSourceLanguageConfig = AutoDetectSourceLanguageConfig.FromLanguages(languages);

  // indicate that we want cognitive services to constantly re-evaluate the current language
  speechConfig.SetProperty(PropertyId.SpeechServiceConnection_LanguageIdMode, "Continuous");

  using var recognizer = new SpeechRecognizer(speechConfig, autoDetectSourceLanguageConfig, audioConfig);

  recognizer.Recognized += (s, e) =>
  {
      // because of our auto detect setup, this will return one of our provided languages
      var autoDetectSourceLanguageResult = AutoDetectSourceLanguageResult.FromResult(e.Result);
      var detectedLanguage = autoDetectSourceLanguageResult.Language;
  };
```

The main caveat to consider for these functionalities is that the actual language support may differ per feature. Belgian Dutch (nl-BE) is for example supported for single recognition purposes but is not a valid target for automatic language detection. Detailed language support is available in the [official documentation](https://learn.microsoft.com/en-us/azure/cognitive-services/Speech-Service/language-support?tabs=stt).

You also have to consider that the auto detection will only consider the provided languages. If the source input is suddenly a whole different language, it will still be detected as one of the languages provided in the configuration.

One of the possible use cases I have in mind, is real-time functionality during video meetings. In order to be able to achieve that, we need to be able to listen to not only our own microphone, but also to our speaker output. The easiest way I found to set this up currently was by creating a virtual microphone on my mac. My current setup uses [BlackHole](https://github.com/ExistentialAudio/BlackHole) so I can listen to both my actual microphone but also forward output to a virtual microphone.

This space is actively being developed by Microsoft. There are some new [transcription APIs](https://learn.microsoft.com/en-gb/azure/cognitive-services/speech-service/conversation-transcription) available which offer functionality out of the box that I'm managing myself right now. This would also offer speaker identification for example out of the box so I'm keen to try this out. Unfortunately, this currently only supports a very specific microphone setup. I'm waiting to get access to the private preview to trial this out on my current setup.

# Doing things with real-time text

So now we can get real-time text events, let's have some fun with it! I built a small application for myself to try out things. The application has an internal event bus to enable subscribing to text events and augment these events with additional info, for example additional analysis context or translations.

The most obvious use case is generating live subtitles so that is also where I started. Here you can make great use of both the 'Recognizing' and the 'Recognized' events. The initial intermediate events already provide great real-time data, when the final event comes in you can easily replace the previous output based on the provided timestamps in the events.

<!-- markdownlint-disable MD033 -->
<video width="750" height="468" controls>
  <source src="/img/demo-subtitles-720.mov" type="video/mp4">
</video>
<!-- markdownlint-enable MD033 -->

When listening to multiple input sources, you want a way to differentiate between both sources. To support this my tool knows the source and will independently update the translation per source and visualize them slightly differently. Due to the way I recorded this demo you can only hear the microphone in the video. The same audio of the previous video is playing at the same time.

<!-- markdownlint-disable MD033 -->
<video width="750" height="468" controls>
  <source src="/img/demo-subtitles-input-sources-720.mov" type="video/mp4">
</video>
<!-- markdownlint-enable MD033 -->

An additional step is live translation. Getting our text is cool, getting the subtitles in the language we prefer is even cooler. In the demo I show both the detected text and the translation, in reality you would probably only turn on the translated version and not show both languages at the same time.

<!-- markdownlint-disable MD033 -->
<video width="750" height="468" controls>
  <source src="/img/demo-translation-720.mov" type="video/mp4">
</video>
<!-- markdownlint-enable MD033 -->

Next to the speech-to-text functionalities, Azure also offers some general language analysis features which I also wanted to experiment with. In the below demo I process every complete sentence to extract key words. When I get the keywords back, the subtitles are updated to underline these. I apologize for the white background in the video which makes it somewhat hard to see the underlining properly.

<!-- markdownlint-disable MD033 -->
<video width="750" height="468" controls>
  <source src="/img/demo-subtitles-key-words-720.mov" type="video/mp4">
</video>
<!-- markdownlint-enable MD033 -->

<br />
<br />
