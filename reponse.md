Here is the complete GraphQL JSON response that Optimizely Content Graph returns for the unified `GetInitialAppData` query when a user navigates to `/benefit` (with `$lob: "BANKING"`, `$pageUrl: "/benefit"`, `$defaultPageUrl: "/dashboard"`):

```json
{
  "data": {
    "redirectRules": {
      "items": []
    },
    "navigationData": {
      "items": [
        {
          "lob": "BANKING",
          "headerMenu": [
            {
              "title": "Benefits & Perks",
              "url": null,
              "targetContent": {
                "_metadata": {
                  "id": "1050",
                  "url": "/benefit"
                },
                "headline": "Comprehensive Health & Wellness Benefits"
              },
              "fallbackMenuItem": null,
              "subItems": [
                {
                  "title": "Medical Coverage",
                  "url": "/benefit/medical",
                  "targetContent": {
                    "_metadata": {
                      "id": "1051",
                      "url": "/benefit/medical"
                    },
                    "headline": "Medical & Dental Plans"
                  },
                  "fallbackMenuItem": null,
                  "subItems": []
                },
                {
                  "title": "Unpublished Sub-menu Item",
                  "url": null,
                  "targetContent": null,
                  "fallbackMenuItem": {
                    "title": "Default Wellness Stipend",
                    "url": "/benefit/wellness-default",
                    "targetContent": {
                      "_metadata": {
                        "id": "1005",
                        "url": "/benefit/wellness-default"
                      },
                      "headline": "Standard Wellness Allowance"
                    }
                  },
                  "subItems": []
                }
              ]
            },
            {
              "title": "Find Care",
              "url": "/find-care",
              "targetContent": {
                "_metadata": {
                  "id": "2010",
                  "url": "/find-care"
                },
                "heroTitle": "Locate In-Network Healthcare Providers"
              },
              "fallbackMenuItem": null,
              "subItems": []
            }
          ],
          "footerMenu": [
            {
              "title": "Terms of Service",
              "url": "/terms",
              "targetContent": {
                "_metadata": {
                  "id": "9001",
                  "url": "/terms"
                },
                "headline": "Terms and Conditions"
              },
              "fallbackMenuItem": null,
              "subItems": []
            },
            {
              "title": "Help Center",
              "url": null,
              "targetContent": null,
              "fallbackMenuItem": {
                "title": "Global Support Portal",
                "url": "/support",
                "targetContent": {
                  "_metadata": {
                    "id": "9002",
                    "url": "/support"
                  }
                }
              },
              "subItems": []
            }
          ]
        }
      ]
    },
    "targetPage": {
      "items": [
        {
          "_metadata": {
            "id": "1050",
            "url": "/benefit",
            "types": ["ArticlePage"]
          },
          "headline": "Comprehensive Health & Wellness Benefits",
          "mainBody": "<p>Welcome to your 2026 employee benefits guide. Below you will find options for medical, dental, vision, and wellness stipends.</p>",
          "author": "HR Benefits Team"
        }
      ]
    },
    "fallbackPage": {
      "items": [
        {
          "_metadata": {
            "id": "1001",
            "url": "/dashboard",
            "types": ["LandingPage"]
          },
          "heroTitle": "Welcome to Your Banking Portal",
          "bannerImage": {
            "url": "/globalassets/banners/dashboard-hero.jpg"
          }
        }
      ]
    },
    "notFoundPage": {
      "items": [
        {
          "_metadata": {
            "id": "4040",
            "url": "/404",
            "types": ["ArticlePage"]
          },
          "headline": "404 - Page Not Found",
          "mainBody": "<p>The page you are looking for does not exist or has been moved.</p>",
          "author": "System Administrator"
        }
      ]
    }
  }
}

```

---

### How Your React App Consumes This Payload

1. **`redirectRules` Check (`items: []`)**:
* Evaluated first. Since the array is empty, no redirect action is taken and rendering continues normally.


2. **`navigationData` Parsing**:
* `resolveMenuTree()` iterates through `headerMenu` and `footerMenu`.
* For *"Unpublished Sub-menu Item"*, `targetContent` is `null`, so the resolver seamlessly picks up `fallbackMenuItem` (`/benefit/wellness-default`).
* The parsed tree is saved into `NavigationContext`.


3. **`targetPage` Resolution (`items[0]`)**:
* The requested `/benefit` page is found in `targetPage.items[0]`.
* The application uses this object to render the view and ignores `fallbackPage` and `notFoundPage`.
