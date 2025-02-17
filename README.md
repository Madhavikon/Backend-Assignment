# Backend Developer Assignment – User Data Management & Twitter OAuth API  

## **Overview**  
This assignment is designed to evaluate your ability to build robust APIs, interact with a MySQL database, and integrate third-party authentication using Twitter OAuth. The assignment consists of two parts:  To know more about as explore our product [ANDROID](https://play.google.com/store/apps/details?id=com.empowerverse.app) || [iOS](https://apps.apple.com/us/app/empowerverse/id6449552284).

1. **User Data Management API** – Handles CSV uploads, user management, and database backup/restore.  
2. **Twitter OAuth Integration** – Implements authentication using Twitter API in a Symfony-based backend.  

## **Technology Stack**  
- **Language**: PHP  
- **Framework**: Symfony  
- **Database**: MySQL  

---

## **Part 1: User Data Management API**  

### **Data Upload & Management**  
You will work with a CSV file (`data.csv`) containing user details with the following columns:  

- `name`  
- `email`  
- `username`  
- `address`  
- `role` (`USER`, `ADMIN`)  

### **API Endpoints to Implement**  

#### **1. Upload and Store Data API**  
- **Endpoint**: `POST /api/upload`  
- **Description**: Allows an admin to upload the `data.csv` file.  
- **Functionality**:  
  - Parses the `data.csv` file.  
  - Saves the data into a database.  
  - Sends an email notification to each user upon successful storage.  
  - Ensures email sending is handled asynchronously (does not block API response).  

#### **2. View Data API**  
- **Endpoint**: `GET /api/users`  
- **Description**: Returns all stored user data from the database.  

#### **3. Backup Database API**  
- **Endpoint**: `GET /api/backup`  
- **Description**: Allows an admin to take a backup of the database.  
- **Functionality**:  
  - Generates a backup file (e.g., `backup.sql`).  

#### **4. Restore Database API**  
- **Endpoint**: `POST /api/restore`  
- **Description**: Allows an admin to restore the database from `backup.sql`.  
- **Functionality**:  
  - Restores the database using the provided backup file.  

### **Email Notification**  
- Use an email service to send notifications to users upon successful data storage.  
- Ensure emails are sent asynchronously.  

---

## **Part 2: Twitter OAuth Integration**  

Your task is to integrate Twitter authentication into the Symfony backend. This includes:  

- Implementing Twitter login using **OAuth 1.0a**.  
- Storing authenticated user details in MySQL.  
- Providing an API endpoint for the mobile app to initiate authentication.  
- Handling the OAuth callback from Twitter and saving user data.  
- Redirecting users back to the app upon successful authentication.  

### **API Endpoints to Implement**  

#### **1. Initiate Twitter Authentication**  
- **Endpoint**: `GET /auth/twitter`  
- **Description**: Redirects the user to Twitter for authentication.  

#### **2. Handle Twitter Callback**  
- **Endpoint**: `GET /auth/twitter/callback`  
- **Description**: Handles the OAuth response, fetches user details, stores them in MySQL, and redirects the user back to the app.  

---

### **Submission Guidelines**  

To successfully complete the assignment, you must submit the following:  

## **Deliverables**  
1. **Symfony Project** with:  
   - Functional **User Data Management API**.  
   - **Twitter OAuth integration** for authentication.  
2. **Database Migration Script** to store user details.  
3. **README.md File** explaining:  
   - How to set up and run the project.  
   - How to configure Twitter API keys.  
   - Example API responses.  
4. **Postman Collection** for testing API endpoints.  
5. **Video Submission**:  
   - **Introduction Video (30-40 sec)**:  
     - Show your face in the video.  
     - Provide a brief introduction about yourself.  
     - Explain what you built in this assignment.  
   - **Screen Recording**:  
     - Run the APIs using **Postman** and show the working endpoints.  
     - Demonstrate how Twitter authentication works.  
     - Show how data is stored and retrieved from the database.  

---

## **Submission Guidelines**  
- Submit your assignment as a **GitHub repository**.  
- Include all files, including the **README.md** and **Postman collection**.  
- Upload the **videos** to Google Drive or YouTube (unlisted) and provide the link in your submission.  
- Notify us upon completion via our **Telegram group**: [Join Here](https://t.me/+I58w50QYkrgwMzc1).  

---

### **Important Notes**  
✅ Ensure that all APIs work correctly before submission.  
✅ Incomplete submissions will not be considered.  
✅ The video submission is mandatory.  

<?php

namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Annotation\Route;
use Doctrine\ORM\EntityManagerInterface;
use App\Entity\User;
use Symfony\Component\Mime\Email;
use Symfony\Component\Mailer\MailerInterface;
use League\Csv\Reader;
use Symfony\Component\HttpFoundation\JsonResponse;
use Abraham\TwitterOAuth\TwitterOAuth;

class UserController extends AbstractController
{
    #[Route('/api/upload', name: 'upload_users', methods: ['POST'])]
    public function upload(Request $request, EntityManagerInterface $entityManager, MailerInterface $mailer): JsonResponse
    {
        $file = $request->files->get('file');
        if (!$file) {
            return new JsonResponse(['error' => 'No file uploaded'], Response::HTTP_BAD_REQUEST);
        }

        $csv = Reader::createFromPath($file->getPathname(), 'r');
        $csv->setHeaderOffset(0);
        $users = [];

        foreach ($csv as $record) {
            $user = new User();
            $user->setName($record['name']);
            $user->setEmail($record['email']);
            $user->setUsername($record['username']);
            $user->setAddress($record['address']);
            $user->setRole($record['role']);
            $entityManager->persist($user);
            $users[] = $user;
        }
        $entityManager->flush();

        foreach ($users as $user) {
            $email = (new Email())
                ->from('noreply@example.com')
                ->to($user->getEmail())
                ->subject('Welcome!')
                ->text('Your data has been stored successfully.');
            $mailer->send($email);
        }

        return new JsonResponse(['message' => 'Users uploaded successfully'], Response::HTTP_OK);
    }

    #[Route('/api/users', name: 'get_users', methods: ['GET'])]
    public function getUsers(EntityManagerInterface $entityManager): JsonResponse
    {
        $users = $entityManager->getRepository(User::class)->findAll();
        return new JsonResponse($users, Response::HTTP_OK);
    }

    #[Route('/auth/twitter', name: 'twitter_login', methods: ['GET'])]
    public function twitterLogin(): Response
    {
        $twitterOAuth = new TwitterOAuth('TWITTER_CONSUMER_KEY', 'TWITTER_CONSUMER_SECRET');
        $requestToken = $twitterOAuth->oauth('oauth/request_token', ['oauth_callback' => 'CALLBACK_URL']);
        
        $url = $twitterOAuth->url('oauth/authorize', ['oauth_token' => $requestToken['oauth_token']]);
        return new JsonResponse(['url' => $url]);
    }

    #[Route('/auth/twitter/callback', name: 'twitter_callback', methods: ['GET'])]
    public function twitterCallback(Request $request, EntityManagerInterface $entityManager): Response
    {
        $oauthToken = $request->query->get('oauth_token');
        $oauthVerifier = $request->query->get('oauth_verifier');

        $twitterOAuth = new TwitterOAuth('TWITTER_CONSUMER_KEY', 'TWITTER_CONSUMER_SECRET');
        $accessToken = $twitterOAuth->oauth('oauth/access_token', ['oauth_verifier' => $oauthVerifier]);

        $user = new User();
        $user->setName($accessToken['screen_name']);
        $user->setUsername($accessToken['screen_name']);
        $user->setEmail($accessToken['screen_name'].'@twitter.com');
        $user->setRole('USER');
        $entityManager->persist($user);
        $entityManager->flush();

        return new JsonResponse(['message' => 'Twitter authentication successful']);
    }
}























Good luck! 🚀
