# pyppeteer-python-pdf-generator

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

## Async Call
```python

from asgiref.sync import sync_to_async

async def async_download_proforma_to_customer(request, proforma_id):
    is_authenticated = await sync_to_async(lambda: request.user.is_authenticated)()
    if not is_authenticated:
        return JsonResponse({"error": "User not authenticated"}, status=400)

    try:
        proforma_header = await ProformaHeader.objects.aget(id=proforma_id)
    except ProformaHeader.DoesNotExist:
        return JsonResponse({"error": "Proforma not found"}, status=400)

    @sync_to_async
    def get_proforma_details() -> QuerySet[ProformaDetail]:
        return ProformaDetail.objects.filter(proformaheader=proforma_header,status=ACTIVE).order_by('lineno')
    @sync_to_async
    def filter_details_count(queryset:QuerySet[ProformaDetail]):
        return queryset.count()
    proforma_details = await get_proforma_details()
    if await filter_details_count(proforma_details) == 0:
        return JsonResponse({'msg': "Please add Items to proforma"}, status=400)
    
    if not proforma_header.confirmed_this_order:
        if not proforma_header.print_order_confirmed:
            proforma_header.print_order_confirmed = True
            proforma_header.sentdate = timezone.now()
            proforma_header.proforma_status = SEND
            proforma_header.asave()

    context = await sync_to_async(proforma_pdf_content)(proforma_header, proforma_details)
    html_content = await sync_to_async(render_to_string)('pshome/proforma/more-option/download.html', context)
    pdf_filename = 'proforma-confirmation'
    pdf_file_url_or_error, gen_status = await generate_pdf(html_content,pdf_filename)
    if gen_status == 400:
        return JsonResponse({'msg':pdf_file_url_or_error}, status=400)
    await sync_to_async(user_log_create)(request, request.user, action="Created", action_message="Proforma Printed confirmed Pdf", module_name='Proforma', module_instance_id=proforma_header.id)   
    return JsonResponse({'file_url':pdf_file_url_or_error,"filename":pdf_filename}, status=200)

```



## Sync Call

```python
from asgiref.sync import async_to_sync


@login_required(login_url='patriot_app:user_login')
def download_proforma_to_customer(request, proforma_id):
    proforma_header = ProformaHeader.objects.filter(id= proforma_id).first()
    proforma_details = ProformaDetail.objects.filter(proformaheader=proforma_header,status=ACTIVE).order_by('lineno')
    
    if not proforma_details:
        return JsonResponse({'msg': "Please add Items to proforma"}, status=400)

    context = proforma_pdf_content(proforma_header, proforma_details)
    template = 'pshome/proforma/more-option/download.html'
    if not proforma_header.confirmed_this_order:
        if not proforma_header.print_order_confirmed:
            proforma_header.print_order_confirmed = True
            proforma_header.sentdate = timezone.now()
            proforma_header.proforma_status = SEND
            proforma_header.save()

    pdf_filename = 'proforma-confirmation.pdf'
    html_content = render_to_string(template, context)
    pdf_file_url_or_error, gen_status = async_to_sync(generate_pdf)(html_content,pdf_filename)
    if gen_status == 400:
        return JsonResponse({'msg':pdf_file_url_or_error}, status=400)
    user_log_create(request, request.user, action="Created", action_message="Proforma Printed confirmed Pdf", module_name='Proforma', module_instance_id=proforma_header.id)   
    return JsonResponse({'file_url':pdf_file_url_or_error,"filename":pdf_filename}, status=200)


```


## How to Download using Js

```javascript


    async function Download() {
        event.preventDefault();
        const spinner = document.getElementById('po-spinner');
        spinner.removeAttribute('style');
        const url = ''
        try {
            const res = await axios.get(url);
            if (res.status === 200) {
                if (res.data.file_url){
                    await downloadFile(res.data.file_url, res.data.filename);
                }
                spinner.style.display= 'none';    
            } else {
                toastr.error(res.data.msg);
                setTimeout(() => {
                    window.location.reload();
                }, 2000); 
            }
        } catch (error) {
            if (error.response){
                toastr.error(error.response.data.msg);
            }
            setTimeout(() => {
                    window.location.reload();
            }, 2000); 
        }
      }


function downloadFile(url, filename) {
    axios({
        url: url,
        method: 'GET',
        responseType: 'blob', // Important
    })
        .then(response => {
            const url = window.URL.createObjectURL(new Blob([response.data]));
            const a = document.createElement('a');
            a.href = url;
            a.download = filename; // Set desired file name here
            document.body.appendChild(a);
            a.click();
            window.URL.revokeObjectURL(url);
            toastr.success(`Successfully Downloaded ${filename}`);
        })
        .catch(error => {
            console.error('Error downloading file:', error);
            toastr.success(error);
        });
}



```
