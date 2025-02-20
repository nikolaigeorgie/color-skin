package main

import (
    "bytes"
    "context"
    "fmt"
    "image"
    "image/jpeg"
    "os"

    "github.com/beezrathashem/kosher-fill/nude"
    "github.com/google/uuid"
    "github.com/aws/aws-sdk-go-v2/aws"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

// Process the image and censor it before uploading to S3
func CensorAndUploadImage(img image.Image, filterType string) (string, error) {
    // Apply the specified filter
    filtered, err := ApplyFilter(img, filterType)
    if err != nil {
        return "", fmt.Errorf("failed to apply filter: %w", err)
    }

    // Convert to JPEG
    var buf bytes.Buffer
    if err := jpeg.Encode(&buf, filtered, &jpeg.Options{Quality: 85}); err != nil {
        return "", fmt.Errorf("failed to encode image: %w", err)
    }

    // Upload to storage
    url, err := UploadToStorage(buf.Bytes())
    if err != nil {
        return "", fmt.Errorf("failed to upload image: %w", err)
    }

    return url, nil
}

// Apply the filter to the image based on the chosen filter type
func ApplyFilter(img image.Image, filterType string) (image.Image, error) {
    detector := nude.NewDetector(img)
    _, err := detector.Parse()
    if err != nil {
        return nil, fmt.Errorf("error parsing image: %w", err)
    }

    switch filterType {
    case "fill":
        return detector.FillSkinPixels(detector.SkinRegions)
    default:
        return detector.DrawImageAndRegions(detector.SkinRegions)
    }
}

// Upload the censored image to S3
func UploadToStorage(imgBytes []byte) (string, error) {
    key := fmt.Sprintf("censored/%s.jpg", uuid.New().String())

    // Upload to S3
    _, err := s3Client.PutObject(context.Background(), &s3.PutObjectInput{
        Bucket:      aws.String(os.Getenv("AWS_BUCKET")),
        Key:         aws.String(key),
        Body:        bytes.NewReader(imgBytes),
        ContentType: aws.String("image/jpeg"),
    })

    if err != nil {
        return "", fmt.Errorf("failed to upload to S3: %w", err)
    }

    // Return the URL
    return fmt.Sprintf("https://%s.s3.amazonaws.com/%s", os.Getenv("AWS_BUCKET"), key), nil
}
