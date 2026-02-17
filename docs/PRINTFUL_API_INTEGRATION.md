# Printful API Integration Blueprint 🎨

## Complete Implementation Guide for Mac Mini Automation

---

## 1. Authentication Setup

### Step 1: Create Printful Account & Get API Token
1. Go to https://developers.printful.com
2. Sign in with your Printful account
3. Navigate to "Private Tokens" section
4. Create new token with scopes:
   - Product Templates (Read/Write)
   - Store Products (Read/Write)
   - Orders (Read)
   - File Library (Read/Write)
5. Save token securely (store in .env file)

### Step 2: Test Authentication
```python
import requests

PRINTFUL_API_TOKEN = "your_token_here"
BASE_URL = "https://api.printful.com"

headers = {
    "Authorization": f"Bearer {PRINTFUL_API_TOKEN}",
    "Content-Type": "application/json"
}

# Test connection
response = requests.get(f"{BASE_URL}/stores", headers=headers)
print(response.json())
```

---

## 2. Mockup Generation Workflow

### Key Printful API Endpoints

#### A. Product Templates API
- `GET /product-templates` - List all templates
- `GET /product-templates/{id}` - Get specific template
- `POST /product-templates` - Create new template
- `PUT /product-templates/{id}` - Update template
- `DELETE /product-templates/{id}` - Delete template

#### B. Mockup Generator API
- `POST /mockup-generator/create-task/{id}` - Generate mockup from template
- `GET /mockup-generator/task` - Check mockup generation status
- Result includes mockup image URLs

### Automated Mockup Generation Flow

```python
import requests
import time
import json

class PrintfulAutomation:
    def __init__(self, api_token):
        self.api_token = api_token
        self.base_url = "https://api.printful.com"
        self.headers = {
            "Authorization": f"Bearer {api_token}",
            "Content-Type": "application/json"
        }

    def upload_design(self, file_path):
        """Upload design file to Printful"""
        with open(file_path, 'rb') as f:
            files = {'file': f}
            response = requests.post(
                f"{self.base_url}/files",
                headers={"Authorization": f"Bearer {self.api_token}"},
                files=files
            )
        return response.json()

    def create_product_template(self, product_id, design_file_id, variant_ids):
        """Create product template with design"""
        payload = {
            "product_id": product_id,
            "variant_ids": variant_ids,
            "files": [
                {
                    "type": "default",
                    "id": design_file_id
                }
            ]
        }
        response = requests.post(
            f"{self.base_url}/product-templates",
            headers=self.headers,
            json=payload
        )
        return response.json()

    def generate_mockups(self, template_id, variant_ids):
        """Generate mockups for product template"""
        payload = {
            "variant_ids": variant_ids,
            "format": "jpg",
            "width": 1000
        }
        response = requests.post(
            f"{self.base_url}/mockup-generator/create-task/{template_id}",
            headers=self.headers,
            json=payload
        )

        task_key = response.json()['result']['task_key']

        # Poll for completion
        while True:
            status_response = requests.get(
                f"{self.base_url}/mockup-generator/task",
                headers=self.headers,
                params={"task_key": task_key}
            )
            status_data = status_response.json()

            if status_data['result']['status'] == 'completed':
                return status_data['result']['mockups']
            elif status_data['result']['status'] == 'failed':
                raise Exception("Mockup generation failed")

            time.sleep(2)

    def bulk_generate_mockups(self, designs, product_variants):
        """Generate mockups for multiple designs and variants"""
        results = []

        for design_path in designs:
            # Upload design
            file_response = self.upload_design(design_path)
            file_id = file_response['result']['id']

            # Create template
            template_response = self.create_product_template(
                product_id=380,  # Gildan 18500 hoodie
                design_file_id=file_id,
                variant_ids=product_variants
            )
            template_id = template_response['result']['id']

            # Generate mockups
            mockups = self.generate_mockups(template_id, product_variants)

            results.append({
                "design": design_path,
                "template_id": template_id,
                "mockups": mockups
            })

        return results

# Usage
api = PrintfulAutomation("your_api_token")
designs = ["design1.png", "design2.png", "design3.png"]
variants = [4012, 4013, 4014]  # Black, White, Navy
mockups = api.bulk_generate_mockups(designs, variants)
```

---

## 3. Product Variant IDs (Gildan 18500 Hoodie)

### Core Colorways for +++ Only
```python
HOODIE_VARIANTS = {
    # Neutral Base
    "Black": 4012,
    "White": 4013,
    "Sport Grey": 4014,
    "Dark Heather": 4015,

    # Bold Colors
    "Navy": 4016,
    "Forest Green": 4017,
    "Maroon": 4018,
    "Purple": 4019,

    # Earth Tones
    "Sand": 4020,
    "Military Green": 4021,
    "Charcoal": 4022,

    # Sizes per color: S, M, L, XL, 2XL, 3XL
}
```

---

## 4. GitHub Integration

### Auto-sync Design Files from GitHub
```python
import os
from github import Github

def sync_designs_from_github(github_token, repo_name, local_path):
    """Pull latest designs from GitHub repo"""
    g = Github(github_token)
    repo = g.get_repo(f"dorianxreese/{repo_name}")

    # Get designs folder contents
    contents = repo.get_contents("designs/product")

    for content in contents:
        if content.path.endswith(('.png', '.jpg', '.svg')):
            # Download file
            file_content = content.decoded_content
            local_file = os.path.join(local_path, content.name)

            with open(local_file, 'wb') as f:
                f.write(file_content)

    return local_path

# Usage
github_token = "your_github_token"
designs_path = sync_designs_from_github(github_token, "plus-only", "./designs")
```

---

## 5. Complete Automation Script

```python
#!/usr/bin/env python3
"""
Printful Mockup Automation
Pulls designs from GitHub, generates mockups, saves back to GitHub
"""

import os
import json
from printful_api import PrintfulAutomation
from github import Github
import requests

# Configuration
PRINTFUL_TOKEN = os.getenv("PRINTFUL_API_TOKEN")
GITHUB_TOKEN = os.getenv("GITHUB_TOKEN")
REPO_NAME = "plus-only"

# Colorways for +++ Only
COLORWAYS = [
    {"name": "Black", "variant_id": 4012},
    {"name": "White", "variant_id": 4013},
    {"name": "Sport Grey", "variant_id": 4014},
    {"name": "Navy", "variant_id": 4016},
    {"name": "Forest Green", "variant_id": 4017},
]

def main():
    print("🚀 +++ Only Mockup Automation")
    print("=" * 60)

    # 1. Pull designs from GitHub
    print("\n📥 Syncing designs from GitHub...")
    designs_path = sync_designs_from_github(GITHUB_TOKEN, REPO_NAME, "./temp_designs")
    design_files = [f for f in os.listdir(designs_path) if f.endswith('.png')]
    print(f"   Found {len(design_files)} designs")

    # 2. Initialize Printful API
    print("\n🎨 Connecting to Printful...")
    printful = PrintfulAutomation(PRINTFUL_TOKEN)

    # 3. Generate mockups for each design + colorway
    print("\n⚙️  Generating mockups...")
    all_mockups = []

    for design in design_files:
        design_path = os.path.join(designs_path, design)
        print(f"\n   Processing: {design}")

        # Upload design
        file_response = printful.upload_design(design_path)
        file_id = file_response['result']['id']
        print(f"      ✅ Uploaded (ID: {file_id})")

        # Generate mockups for all colorways
        variant_ids = [c["variant_id"] for c in COLORWAYS]
        mockups = printful.generate_mockups(file_id, variant_ids)

        all_mockups.append({
            "design": design,
            "mockups": mockups
        })

        print(f"      ✅ Generated {len(mockups)} mockups")

    # 4. Save mockups back to GitHub
    print("\n💾 Saving mockups to GitHub...")
    save_mockups_to_github(GITHUB_TOKEN, REPO_NAME, all_mockups)

    print("\n✅ COMPLETE! All mockups generated and saved.")
    print(f"   View at: https://github.com/dorianxreese/{REPO_NAME}/tree/main/designs/mockups")

if __name__ == "__main__":
    main()
```

---

## 6. Mac Mini Setup Instructions

### Phase 1: Essential Setup (Tonight)
```bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Python & dependencies
brew install python3 git

# Install required packages
pip3 install requests PyGithub python-dotenv

# Clone repos
cd ~/Projects
git clone https://github.com/dorianxreese/plus-only.git
cd plus-only

# Set up environment variables
echo "PRINTFUL_API_TOKEN=your_token_here" > .env
echo "GITHUB_TOKEN=your_token_here" >> .env
```

### Phase 2: Automation Setup (This Week)
```bash
# Install automation platform
brew install n8n

# Set up scheduled automation
crontab -e
# Add: 0 9 * * * cd ~/Projects/plus-only && python3 automation/printful/generate_mockups.py
```

---

## 7. Integration with Multi-Agent System

### Design Agent → Printful Automation
```python
# Design Agent creates new design
# → Saves to GitHub (designs/product/)
# → Triggers webhook
# → Operations Agent runs mockup generation
# → Saves mockups to GitHub (designs/mockups/)
# → Marketing Agent gets notified, pulls mockups for social
```

---

## 8. Next Steps

### Immediate (Tonight - Mac Mini Setup)
1. Install Homebrew, Python, Git
2. Clone plus-only repo
3. Create Printful developer account
4. Generate API token
5. Test API connection

### This Week
1. Upload 3 selected designs to GitHub
2. Run first automated mockup generation
3. Review mockup quality
4. Set up n8n for workflow automation

### This Month
1. Full multi-agent integration
2. Automated social media mockup posting
3. Drop campaign automation
4. Analytics dashboard

---

## Resources
- [Printful API Docs](https://developers.printful.com/docs)
- [Product Templates API](https://developers.printful.com/docs/#tag/Product-Templates-API)
- [Mockup Generator API](https://developers.printful.com/docs/#tag/Mockup-Generator-API)
- [GitHub API - PyGithub](https://pygithub.readthedocs.io/)

---

*Blueprint created for DorianxReese +++ Only Brand*
*Last updated: Feb 17, 2026*
