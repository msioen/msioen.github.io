---
layout: post
title: Converting speech to images with Azure OpenAI
subtitle: Azure AI Experiments
share-img: https://michielsioen.be/img/social-share-speech-to-image.png
---

When I started playing around with [speech-to-text functionalities](/2023-05-13-ai-speech-to-value/), the goal was to do some fun things with the captured speech. Wouldn't it be fun if you create your own fairy tales for your kids and while you're talking you get matching illustrations? To get some nice background graphics while playing some dungeons and dragons? Or even giving a presentation without preparing any slides? In this post I'll describe my initial attempts to make this happen, let's have some fun!

# Azure OpenAI

[Azure OpenAI Service](https://azure.microsoft.com/en-ca/products/cognitive-services/openai-service) is the offering from Azure to get access to OpenAI's powerful language models, these are the language models which are also running behind tools like ChatGPT. Compared to directly using OpenAI, Azure adds the enterprise features you would expect: security capabilities, private networking and regional availability.

You do need to apply to get access to Azure OpenAI, all the necessary information to apply can be found on the [official documentation](https://learn.microsoft.com/en-us/azure/cognitive-services/openai/overview#how-do-i-get-access-to-azure-openai).

Once you have access to the service, the [Azure AI Studio](https://oai.azure.com/portal) is a great tool to get started. It showcases all Azure OpenAI features, quickly experiment with language completions, chat and image generation.

![Azure AI Studio](/img/azure-ai-studio.png)

# Prompting for completions

I'm using the completions endpoint as my interface to the language model. This works by entering your input text as the prompt. The model will then complete your text or attempt to match the pattern you gave it. Setting up a good prompt is key to get good results.

Using Azure OpenAI services in C# is straightforward with the [official NuGet package](https://www.nuget.org/packages/Azure.AI.OpenAI).

```csharp
  var client = new OpenAIClient(
      new Uri($"https://{azureResourceName}.openai.azure.com/"),
      new AzureKeyCredential(azureApiKey));

  var response = await client.GetCompletionsAsync("gpt-35-turbo", 
    new CompletionsOptions()
    {
        Prompts = { prompt },
        MaxTokens = 60,
        Temperature = 0.5f
    });

  var result = response.Value.Choices[0].Text;
```

So far, I haven't found the ideal prompt yet for my use case. The context I'm currently providing the model is real-time text which may or may not be complete sentences. Tricking the system into actually summarizing and extracting keywords, instead of completing the partial context isn't always straightforward. My original goal was even to let the language model create the final image prompt fully itself. I wasn't able to craft a prompt to let GPT do this completely on its own yet, but I still have some ideas to validate.

# Generating images

The art and science of generating images through artificial intelligence is still relatively new. This means that nobody has the best instructions yet, showing how to consistently create amazing images. A great resource is the [DALL-E 2 Prompt Book](https://dallery.gallery/the-dalle-2-prompt-book/). Even if you're not planning to generate images, this is still a fun read as it gives a broad overview of art styles and history.

DALL-E on Azure is currently still in private preview. The [python SDK](https://github.com/openai/openai-python) does have preview support in the library, other SDKs do not have support yet. At time of writing, it is also only available in the East US region.

After getting access to the private review, I received some basic pdf documentation which, in combination with the Azure AI Studio examples, was enough to build my own implementation which you can find below. Note that, as the API is still in preview, the actual data formats are still likely to change.

```csharp
  
  _httpClient = new HttpClient();
  _httpClient.BaseAddress = new Uri($"https://{resourceName}.openai.azure.com/");
        
  _httpClient.DefaultRequestHeaders.TryAddWithoutValidation("api-key", apiKey);
  _httpClient.DefaultRequestHeaders.TryAddWithoutValidation("Content-Type", "application/json");

  public async Task<AzureDalleImageCreateResponse?> CreateImage(
    string prompt,
    string resolution = "1024x1024",
    CancellationToken cancellationToken = default)
  {
      if (string.IsNullOrWhiteSpace(prompt))
      {
          return null;
      }
      
      // 1. launch create request
      //POST https://{aoai-resource}.openai.azure.com/dalle/text-to-image?api-version=2022-08-03-preview
      var request = new AzureDalleImageCreateRequest()
      {
          Prompt = prompt,
          Size = resolution,
      };
      
      var requestUri = "openai/images/generations:submit?api-version=2023-06-01-preview";
      var response = await _httpClient.PostAsJsonAsync(requestUri, request, cancellationToken);
      var location = response.Headers
          .GetValues("operation-location")
          .First();
      
      var retryAfter = "1";
      if (response.Headers.TryGetValues("retry-after", out var retryAfterHeaders))
      {
          retryAfter = retryAfterHeaders.First();
      }
      
      // 2. wait for indicated time
      AzureDalleImageCreateResponse? result = null;
      while (result?.Status is not ("succeeded" or "failed"))
      {
          await Task.Delay(TimeSpan.FromSeconds(int.Parse(retryAfter)), cancellationToken);
          response = await _httpClient.GetAsync(location, cancellationToken);

          result = await response.Content.ReadFromJsonAsync<AzureDalleImageCreateResponse>(cancellationToken);
      }

      return result;
  }
```

# Visualising stories

Our goal was to create real-time illustrations for a story so that is what we're going to do. The story which is used in this demo was generated with Azure OpenAI in the completions playground. I used the Speech Studio to read it out aloud in real-time while generating the images.

As part of my previous blogpost, I already created a tool which listens to speech and can show real-time subtitles. I built further on this tool to add this story visualisation layer. It works as follows:

- transcribe speech to text using the Azure Speech SDK
- constantly gather the detected text
- decide if there is enough context to start generating images, and if enough time has lapsed
- prompt Azure OpenAI with the context of the last 30 seconds to provide keywords
- generate a DALL-E prompt based on these keywords and trigger image creation

To keep the page load small, I added the demo below as a gif. The full experience, including sound, can be checked out [here](/img/speech-to-image.mov).

![Demo](/img/speech-to-image.gif)

<br />
<br />
