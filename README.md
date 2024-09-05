using System;
using System.Drawing;
using System.Drawing.Imaging;
using System.IO;

namespace DigitalSteganography
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Digital Steganography Demo");

            // 1. Choose an action (Hide or Retrieve)
            Console.WriteLine("Choose an action:");
            Console.WriteLine("1. Hide data in an image");
            Console.WriteLine("2. Retrieve data from an image");
            int choice = Convert.ToInt32(Console.ReadLine());

            // 2. Handle Hiding or Retrieving
            if (choice == 1)
            {
                HideData();
            }
            else if (choice == 2)
            {
                RetrieveData();
            }
            else
            {
                Console.WriteLine("Invalid choice.");
            }
        }

        // Hide data in an image
        static void HideData()
        {
            Console.WriteLine("Enter the path to the image file:");
            string imagePath = Console.ReadLine();

            Console.WriteLine("Enter the secret message to hide:");
            string secretMessage = Console.ReadLine();

            // Convert message to bytes
            byte[] messageBytes = System.Text.Encoding.ASCII.GetBytes(secretMessage);

            // Hide data in the image
            HideDataInImage(imagePath, messageBytes);

            Console.WriteLine("Data hidden successfully.");
        }

        // Retrieve data from an image
        static void RetrieveData()
        {
            Console.WriteLine("Enter the path to the image file:");
            string imagePath = Console.ReadLine();

            // Retrieve hidden data
            string retrievedMessage = RetrieveDataFromImage(imagePath);

            if (retrievedMessage != null)
            {
                Console.WriteLine($"Retrieved message: {retrievedMessage}");
            }
            else
            {
                Console.WriteLine("No data found in the image.");
            }
        }

        // Hide data in an image (LSB method)
        static void HideDataInImage(string imagePath, byte[] messageBytes)
        {
            Bitmap image = new Bitmap(imagePath);
            BitmapData bitmapData = image.LockBits(new Rectangle(0, 0, image.Width, image.Height), ImageLockMode.ReadWrite, PixelFormat.Format24bppRgb);

            // Get pointer to the image data
            unsafe
            {
                byte* ptr = (byte*)bitmapData.Scan0;

                // Insert message bytes into LSBs of image pixels
                int messageIndex = 0;
                for (int y = 0; y < image.Height; y++)
                {
                    for (int x = 0; x < image.Width; x++)
                    {
                        if (messageIndex < messageBytes.Length)
                        {
                            // Modify the last bit of each color channel
                            ptr[y * bitmapData.Stride + x * 3] = (byte)((ptr[y * bitmapData.Stride + x * 3] & 254) | (messageBytes[messageIndex] & 1));
                            ptr[y * bitmapData.Stride + x * 3 + 1] = (byte)((ptr[y * bitmapData.Stride + x * 3 + 1] & 254) | ((messageBytes[messageIndex] >> 1) & 1));
                            ptr[y * bitmapData.Stride + x * 3 + 2] = (byte)((ptr[y * bitmapData.Stride + x * 3 + 2] & 254) | ((messageBytes[messageIndex] >> 2) & 1));
                            messageIndex++;
                        }
                    }
                }
            }

            // Unlock the image bits
            image.UnlockBits(bitmapData);

            // Save the modified image
            image.Save(imagePath + "_hidden.png", ImageFormat.Png);
        }

        // Retrieve data from an image (LSB method)
        static string RetrieveDataFromImage(string imagePath)
        {
            Bitmap image = new Bitmap(imagePath);
            BitmapData bitmapData = image.LockBits(new Rectangle(0, 0, image.Width, image.Height), ImageLockMode.ReadWrite, PixelFormat.Format24bppRgb);

            // Get pointer to the image data
            unsafe
            {
                byte* ptr = (byte*)bitmapData.Scan0;

                List<byte> retrievedBytes = new List<byte>();
                int messageIndex = 0;
                for (int y = 0; y < image.Height; y++)
                {
                    for (int x = 0; x < image.Width; x++)
                    {
                        // Extract bits from LSBs
                        byte retrievedByte = (byte)((ptr[y * bitmapData.Stride + x * 3] & 1) |
                                                 ((ptr[y * bitmapData.Stride + x * 3 + 1] & 1) << 1) |
                                                 ((ptr[y * bitmapData.Stride + x * 3 + 2] & 1) << 2));

                        retrievedBytes.Add(retrievedByte);
                        messageIndex++;
                    }
                }

                // Convert retrieved bytes back to a string
                if (retrievedBytes.Count > 0)
                {
                    return System.Text.Encoding.ASCII.GetString(retrievedBytes.ToArray());
                }
            }

            // Unlock the image bits
            image.UnlockBits(bitmapData);

            return null; // No data found
        }
    }
}
