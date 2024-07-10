# Using S3 bucket to store OG images in AWS Amplify app

The website I'm working on, [Mi Home](https://www.mi-home.pro) is hosted by AWS Amplify, it uses Next.js framework and uses S3 bucket to store images. 

Everything is fine for that, except one thing. Amazon S3 storage, where images are saved, is not public, and to get temporary image URL I using the code like this:
```typescript
finalPath =  await Storage.get(path, {
            level: "public",
            expires: 7*24*3600
        })
```
It produces very long links like ` https://mihomeproimages154242-production.s3.eu-west-3.amazonaws.com/public/goods/yeelight_1s_1se_colorful_bulb_e27_smart_app_wifi_r.webp?x-amz-content-sha256=e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855&x-amz-user-agent=aws-amplify%2F5.3.18+storage%2F2+framework%2F102&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=ASIAYV5MP7SRPHSND5ZZ%2F20240710%2Feu-west-3%2Fs3%2Faws4_request&X-Amz-Date=20240710T080453Z&X-Amz-SignedHeaders=host&X-Amz-Expires=604800&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEHgaCWV1LXdlc3QtMyJHMEUCIQCj1SRy2ybvFSKlkDkAJ2SmqLGZ5BlA%2Bv8jQju5otLVTQIgHoRDVZ5FQY8Sn9EldJd%2BS%2FYMFQH8jP9...` 

(this is just a fragment). Ok, that approach works fine with `<Image />` component of Next.js, but I faced a problem when start using NextSeo component:
```typescript
<NextSeo
        title={params.name}
        description={params.annotationTxt}
        additionalMetaTags = {
        [{
          name: 'keywords',
          content: params?.keywords ? params.keywords : '' 
        },]
      }
        openGraph={{ type: 'website',
          url: domain,
          title: params.name,
          description: params.annotationTxt,
          images: [ // .jpg 640x640
            {
              url: params.finalPath,
              width: 640, height: 640, alt: params.name,
            }
          ]
      }}
      />
```

Link for OG image got broken, and images for each webpage did not show. It was even not NextSeo problem, I'll skip this topic for now. 

After some thought, it was decided to use separate public S3 bucket and store OG images there. 

I was create a new bucket named "mi-home-pro-img" in the project AWS account, to make it public need go to "Permissions" tab in the bucket, turn off "Block all public access" option, and add bucket policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicRead",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::mi-home-pro-img/*"
        }
    ]
}
```

After that I have modified function which retrieves the path for OG image. It checks is that image already exists in the new S3 bucket, and if no - it will read the good image from protected S3 bucjet, resize loaded image to 640x640, convert to JPEG format, and store new image into the new bucket.

Here is simplified part of that function (protected bucket was added as Storage as part of Amplify app, but new bucket was added from AWS console, so they accessed a bit differently):

```typescript
import {Storage} from "aws-amplify"
import sharp from 'sharp'
import axios from 'axios'
import { PutObjectCommand, S3Client } from '@aws-sdk/client-s3'

export const getUploadedGoodsImage = async (name: string): Promise<string> => {
const bucketName = 'mi-home-pro-img'
        const fileName = `goods/${name}.jpg`
        const url = `https://${bucketName}.s3.eu-west-3.amazonaws.com/${fileName}`
        let response
        try {
            response = await axios.get(url, {responseType: 'arraybuffer'})
        } catch (e) {
            console.log('Image is not exist at img bucket, need to re-create')
        }

        console.log('>>> Img response status:', response?.status)
        if (response?.status === 200) {
            console.log('Return existing image:', url)
            return url
        }
        console.log('Create resized image and store into S3:')
        finalPath =  await Storage.get(path, {
            level: "public",
            expires: 120
        })
        try {
            response = await axios.get(finalPath, {responseType: 'arraybuffer'})
        } catch (e) {
            console.log('Error: Main S3 Image is not exist')
        }
        if (response?.status === 200) {
            console.log('Image was read from the main S3...')

            const s3 = new S3Client({
                region: 'eu-west-3',
                credentials: {
                    accessKeyId: process.env.ACCESS_KEY_ID as string,
                    secretAccessKey: process.env.SECRET_ACCESS_KEY as string,
                },
            })
            
            const output = await sharp(response.data).resize({
                fit: sharp.fit.contain,
                width: 640,
                height: 640
            })
              .toFormat('jpeg')
              .jpeg({quality: 80, force: true})
              .toBuffer()
              .then(data => { // write to "img" S3 bucket:
                  const command = new PutObjectCommand({
                      Bucket:  bucketName,
                      Key: fileName,
                      Body: data,
                      ContentType: 'image/jpeg'
                  })
                  try {
                      const response = s3.send(command).then(resp => {
                          console.log('>> Uploaded to S3!');
                      })
                  } catch (err) {
                      console.error('# Error on write to S3:', err)
                  }
              })
        }
        console.log(`All done! New URL: ${url}`)
        return url
}
```

Result: go to any Good page in the website (e.g. this: [Smart Human Body Sensor 2S](https://www.mi-home.pro/goods/xiaomi_mijia_smart_human_body_sensor_2s_high_sensi), press Ctrl+U to view source, search there for "og:image" - `<meta property="og:image" content="https://mi-home-pro-img.s3.eu-west-3.amazonaws.com/goods/xiaomi_mijia_smart_human_body_sensor_2s_high_sensi.jpg"/><meta property="og:image:alt" content="Xiaomi Mijia Smart Human Body Sensor 2S"/>`, and you can see, what OG image saved correctly and available for reading:

![Example of stored OG image](https://mi-home-pro-img.s3.eu-west-3.amazonaws.com/goods/xiaomi_mijia_smart_human_body_sensor_2s_high_sensi.jpg)


