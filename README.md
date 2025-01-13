# pyppeteer-python-pdf-genarater

```python

from pyppeteer import launch
import os
from typing import List, Union, Tuple, Dict

from APP import settings
from app.s3 import upload_file_to_s3
from app.utils import delete_file_in_directory
from asgiref.sync import sync_to_async

async def generate_pdf(html_content : str, pdf_name : str) -> Tuple[str,int]:
    try:
        os.makedirs("media/", exist_ok=True)
        output_file_path = f'media/{pdf_name}'

        background_css = """
            @page {
                background-color: #FFFFFF;
                margin: 0;
            }
        """

        browser = await launch(
            handleSIGINT=False,
            handleSIGTERM=False,
            handleSIGHUP=False,
            headless=True,
            args=['--no-sandbox', '--disable-dev-shm-usage']
        )

        page = await browser.newPage()
        await page.setContent(html_content)
        await page.addStyleTag(content=background_css)

        await page.pdf({
            'path': output_file_path,
            'format': 'Letter',
            'printBackground': True,
            'margin': {'top': '0', 'right': '0', 'bottom': '0', 'left': '0'},  # Explicit margins
        })

        await browser.close()
        await sync_to_async(upload_file_to_s3)(f"pdf/{pdf_name}", output_file_path)
        await sync_to_async(delete_file_in_directory)(output_file_path)        
        return f"{settings.BASE_IMAGE_URL}/pdf/{pdf_name}", 200   # Return the file path of the generated PDF
    except Exception as e:
        print(f"Error generating PDF: {e}")
        raise
```

