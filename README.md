# Procurement Suite Configuration

This repository contains configuration files for different deployments of the Procurement Suite application.

## Overview

The configuration files in this repository control various UI features, branding elements, and functionality options for different instances of the Procurement Suite.

## Configuration Files

### [4c_config.json](4c_config.json)

Configuration for the 4C deployment with the following customizations:
- Custom header logo
- Partner logos on the homepage
- Feature toggles for various application components

## Configuration Structure

Each configuration file follows this general structure:

```json
{
  "header_logo": "URL to the header logo image",
  "homepage_logos": [
    {
      "url": "Partner website URL",
      "logo": "Partner logo image URL"
    }
  ],
  "hide_feature_name": boolean
}
```

### Feature Toggles

The following feature toggles are available:
- `hide_chat`: Toggles the chat functionality
- `hide_transcription`: Toggles the transcription functionality
- `hide_bullseye`: Toggles the bullseye functionality
- `hide_service_savings`: Toggles the service savings functionality
- `hide_sentiment_analysis`: Toggles the sentiment analysis functionality
- `hide_retail_analytics`: Toggles the retail analytics functionality

## Usage

To use a configuration file, reference it in your deployment settings or environment variables according to your deployment setup.

## Adding New Configurations

When adding new configuration files, follow the existing pattern and ensure all required properties are included.