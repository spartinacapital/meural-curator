# meural-curator

Natural language photo curation for [Netgear Meural](https://www.netgear.com/home/digital-art-canvas/) digital picture frames. Uses Claude AI to interpret plain English requests, queries your Apple Photos library, lets you preview selections, and uploads to your Meural frame.

## How it works

1. You type a natural language prompt describing the album you want
2. Claude AI translates that into a structured query against your Apple Photos library
3. Photos are selected using face recognition, dates, labels, quality scores, and more
4. A preview album is created in Apple Photos so you can remove any you don't like
5. Approved photos are exported and uploaded to your Meural frame via its REST API

## Requirements

- macOS (uses Apple Photos library and PhotoScript)
- Python 3.11+ (installed via Homebrew)
- An [Anthropic API key](https://console.anthropic.com/) for Claude
- A Netgear Meural account and frame

## Setup

```bash
# Clone the repo
git clone https://github.com/spartinacapital/meural-curator.git
cd meural-curator

# Create virtual environment and install dependencies
python3.11 -m venv .venv
source .venv/bin/activate
pip install osxphotos photoscript anthropic requests boto3

# Make the script executable
chmod +x meural

# Optionally symlink for global access
sudo ln -sf "$(pwd)/meural" /opt/homebrew/bin/meural
```

## Configuration

On first run, meural-curator will interactively prompt you for:

1. **Meural account email** (required)
2. **Names & birthdates** (optional — enables age-based photo queries)
3. **Anthropic API key** — stored in `.credentials.json`
4. **Meural password** — stored in `.credentials.json`

Your answers are saved to `config.json` and `.credentials.json` (both gitignored). See `config.example.json` for the format.

## Usage

```bash
# Basic usage - describe the album you want in plain English
meural "Create an album with 150 photos of the kids at ages 0-3"

meural "Create an album of Christmas pictures over the years"

meural "50 best landscape photos from summer 2024"

meural "Family beach vacation photos"
```

### Options

```
meural "your prompt"              # Full flow: plan → select → preview → upload
meural "your prompt" --dry-run    # Plan and select photos but don't upload
meural "your prompt" --show-plan  # Show the AI-generated query plan
meural "your prompt" --no-preview # Skip preview, upload directly
meural --list-devices             # List your Meural devices
meural --list-galleries           # List your Meural galleries
meural "your prompt" --device ID  # Specify which Meural device to load
```

### Preview workflow

By default, meural-curator creates a temporary album in Apple Photos called "Meural Preview - {album name}". Open Photos, review the album, and delete any photos you don't want. When you return to the terminal and confirm, only the remaining photos are uploaded. The preview album is automatically cleaned up.

## How photo selection works

- **Face recognition**: Apple Photos' built-in person detection identifies who's in each photo
- **Newborn matching**: For babies under 6 months (where face recognition often fails), the tool finds photos with Apple's "Baby" label near the child's birthdate
- **Quality scores**: Apple Photos assigns ML-based aesthetic scores (0-1) to every photo. The tool uses these to pick the best shots
- **Time distribution**: When selecting N photos from a large pool, photos are divided into equal time buckets and the highest-scored photo from each bucket is chosen, giving even temporal coverage
- **Orientation filtering**: Request landscape-only photos for best display on horizontally mounted frames
- **Holiday/label detection**: Apple Photos auto-detects holidays (Christmas, Thanksgiving, etc.) and scene labels (Beach, Mountain, Snow, etc.) which can be used for thematic albums

## Meural API

The tool authenticates via AWS Cognito (same as the Meural mobile app) and uses the Meural REST API at `api.meural.com/v0/` to create galleries, upload images, and load galleries onto devices. Uploads include retry logic for flaky 502/504 responses and periodic re-authentication to handle token expiry during long uploads.
