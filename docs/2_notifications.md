---
title: Notifications
layout: default
nav_order: 2
---

# Notifications with Firebase

Hey team! Let's implement push notifications in our NestJS backend using Firebase Cloud Messaging (FCM). This guide will help you set up a robust notification system that can send messages to web and mobile clients.

## Overview

Firebase Cloud Messaging (FCM) provides a reliable way to send notifications from your NestJS backend to clients across different platforms. This implementation will cover both sending notifications and managing user tokens.

## Setup

### 1. Firebase Project Configuration

First, set up your Firebase project:

1. Go to the [Firebase Console](https://console.firebase.google.com/)
2. Create a new project or select an existing one
3. Go to Project Settings > Service Accounts
4. Generate a new private key (downloads a JSON file)
5. Store this file securely in your project

### 2. Install Dependencies

```bash
npm install firebase-admin
npm install @nestjs/config
```

### 3. Environment Configuration

Add Firebase configuration to your `.env` file:

```env
FIREBASE_PROJECT_ID=your-project-id
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
FIREBASE_CLIENT_EMAIL=firebase-adminsdk-xxxxx@your-project.iam.gserviceaccount.com
```

### 4. Firebase Module Setup

Create a Firebase module for your NestJS application:

```typescript
// firebase/firebase.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { FirebaseService } from './firebase.service';

@Module({
  imports: [ConfigModule],
  providers: [FirebaseService],
  exports: [FirebaseService],
})
export class FirebaseModule {}
```

### 5. Firebase Service

```typescript
// firebase/firebase.service.ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as admin from 'firebase-admin';

@Injectable()
export class FirebaseService implements OnModuleInit {
  onModuleInit() {
    if (!admin.apps.length) {
      admin.initializeApp({
        credential: admin.credential.cert({
          projectId: this.configService.get('FIREBASE_PROJECT_ID'),
          privateKey: this.configService.get('FIREBASE_PRIVATE_KEY').replace(/\\n/g, '\n'),
          clientEmail: this.configService.get('FIREBASE_CLIENT_EMAIL'),
        }),
      });
    }
  }

  constructor(private configService: ConfigService) {}

  async sendNotification(token: string, notification: {
    title: string;
    body: string;
    imageUrl?: string;
  }, data?: Record<string, string>) {
    try {
      const message = {
        token,
        notification: {
          title: notification.title,
          body: notification.body,
          imageUrl: notification.imageUrl,
        },
        data,
        android: {
          priority: 'high',
          notification: {
            sound: 'default',
            channelId: 'default',
          },
        },
        apns: {
          payload: {
            aps: {
              sound: 'default',
            },
          },
        },
      };

      const response = await admin.messaging().send(message);
      console.log('Successfully sent message:', response);
      return response;
    } catch (error) {
      console.error('Error sending message:', error);
      throw error;
    }
  }

  async sendMulticastNotification(tokens: string[], notification: {
    title: string;
    body: string;
    imageUrl?: string;
  }, data?: Record<string, string>) {
    try {
      const message = {
        tokens,
        notification: {
          title: notification.title,
          body: notification.body,
          imageUrl: notification.imageUrl,
        },
        data,
        android: {
          priority: 'high',
          notification: {
            sound: 'default',
            channelId: 'default',
          },
        },
        apns: {
          payload: {
            aps: {
              sound: 'default',
            },
          },
        },
      };

      const response = await admin.messaging().sendMulticast(message);
      console.log('Successfully sent messages:', response);
      return response;
    } catch (error) {
      console.error('Error sending multicast message:', error);
      throw error;
    }
  }

  async sendTopicNotification(topic: string, notification: {
    title: string;
    body: string;
    imageUrl?: string;
  }, data?: Record<string, string>) {
    try {
      const message = {
        topic,
        notification: {
          title: notification.title,
          body: notification.body,
          imageUrl: notification.imageUrl,
        },
        data,
        android: {
          priority: 'high',
          notification: {
            sound: 'default',
            channelId: 'default',
          },
        },
        apns: {
          payload: {
            aps: {
              sound: 'default',
            },
          },
        },
      };

      const response = await admin.messaging().send(message);
      console.log('Successfully sent topic message:', response);
      return response;
    } catch (error) {
      console.error('Error sending topic message:', error);
      throw error;
    }
  }
}
```

## Implementation

### ⚠️ **IMPORTANTE: Gestión de Tokens en Base de Datos**

**Los tokens FCM DEBEN ser almacenados en la base de datos** para poder enviar notificaciones a usuarios específicos. Sin esto, no podrás enviar notificaciones personalizadas.

### 1. User Token Management

Create an entity to store user FCM tokens:

```typescript
// users/entities/user-token.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, CreateDateColumn, UpdateDateColumn } from 'typeorm';

@Entity('user_tokens')
export class UserToken {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  userId: string;

  @Column()
  fcmToken: string;

  @Column({ default: true })
  isActive: boolean;

  @Column({ nullable: true })
  deviceType: string; // 'web', 'ios', 'android'

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

### 2. Notification Service

```typescript
// notifications/notifications.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { FirebaseService } from '../firebase/firebase.service';
import { UserToken } from '../users/entities/user-token.entity';

@Injectable()
export class NotificationsService {
  constructor(
    @InjectRepository(UserToken)
    private userTokenRepository: Repository<UserToken>,
    private firebaseService: FirebaseService,
  ) {}

  async saveUserToken(userId: string, fcmToken: string, deviceType?: string) {
    // Check if token already exists
    const existingToken = await this.userTokenRepository.findOne({
      where: { fcmToken, isActive: true },
    });

    if (existingToken) {
      // Update existing token
      existingToken.userId = userId;
      existingToken.deviceType = deviceType;
      existingToken.updatedAt = new Date();
      return await this.userTokenRepository.save(existingToken);
    }

    // Create new token
    const userToken = this.userTokenRepository.create({
      userId,
      fcmToken,
      deviceType,
      isActive: true,
    });

    return await this.userTokenRepository.save(userToken);
  }

  async removeUserToken(fcmToken: string) {
    const token = await this.userTokenRepository.findOne({
      where: { fcmToken, isActive: true },
    });

    if (token) {
      token.isActive = false;
      await this.userTokenRepository.save(token);
    }
  }

  async sendNotificationToUser(userId: string, notification: {
    title: string;
    body: string;
    imageUrl?: string;
  }, data?: Record<string, string>) {
    const userTokens = await this.userTokenRepository.find({
      where: { userId, isActive: true },
    });

    if (userTokens.length === 0) {
      throw new Error('No active tokens found for user');
    }

    const tokens = userTokens.map(ut => ut.fcmToken);
    return await this.firebaseService.sendMulticastNotification(tokens, notification, data);
  }

  async sendNotificationToUsers(userIds: string[], notification: {
    title: string;
    body: string;
    imageUrl?: string;
  }, data?: Record<string, string>) {
    const userTokens = await this.userTokenRepository.find({
      where: { userId: { $in: userIds }, isActive: true },
    });

    if (userTokens.length === 0) {
      throw new Error('No active tokens found for users');
    }

    const tokens = userTokens.map(ut => ut.fcmToken);
    return await this.firebaseService.sendMulticastNotification(tokens, notification, data);
  }

  async sendNotificationToTopic(topic: string, notification: {
    title: string;
    body: string;
    imageUrl?: string;
  }, data?: Record<string, string>) {
    return await this.firebaseService.sendTopicNotification(topic, notification, data);
  }
}
```

### 🔄 **Flujo Completo de Gestión de Tokens**

1. **Cliente genera token**: El frontend (React/React Native) solicita permisos y genera un token FCM
2. **Cliente envía token**: El frontend envía el token al endpoint `/notifications/token`
3. **Backend guarda token**: NestJS almacena el token en la base de datos asociado al usuario
4. **Backend envía notificación**: Cuando necesitas enviar una notificación, NestJS:
   - Busca los tokens activos del usuario en la base de datos
   - Envía la notificación usando esos tokens
   - Firebase entrega la notificación al dispositivo

**Sin este flujo, no podrás enviar notificaciones personalizadas a usuarios específicos.**
```

### 3. Notification Controller

```typescript
// notifications/notifications.controller.ts
import { Controller, Post, Body, UseGuards, Get, Param } from '@nestjs/common';
import { NotificationsService } from './notifications.service';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { RolesGuard } from '../auth/guards/roles.guard';
import { Roles } from '../auth/decorators/roles.decorator';

@Controller('notifications')
@UseGuards(JwtAuthGuard, RolesGuard)
export class NotificationsController {
  constructor(private notificationsService: NotificationsService) {}

  @Post('token')
  async saveToken(
    @Body() body: { fcmToken: string; deviceType?: string },
    @Request() req,
  ) {
    return await this.notificationsService.saveUserToken(
      req.user.id,
      body.fcmToken,
      body.deviceType,
    );
  }

  @Post('send/user/:userId')
  @Roles('admin')
  async sendToUser(
    @Param('userId') userId: string,
    @Body() body: {
      title: string;
      body: string;
      imageUrl?: string;
      data?: Record<string, string>;
    },
  ) {
    return await this.notificationsService.sendNotificationToUser(
      userId,
      {
        title: body.title,
        body: body.body,
        imageUrl: body.imageUrl,
      },
      body.data,
    );
  }

  @Post('send/users')
  @Roles('admin')
  async sendToUsers(
    @Body() body: {
      userIds: string[];
      title: string;
      body: string;
      imageUrl?: string;
      data?: Record<string, string>;
    },
  ) {
    return await this.notificationsService.sendNotificationToUsers(
      body.userIds,
      {
        title: body.title,
        body: body.body,
        imageUrl: body.imageUrl,
      },
      body.data,
    );
  }

  @Post('send/topic/:topic')
  @Roles('admin')
  async sendToTopic(
    @Param('topic') topic: string,
    @Body() body: {
      title: string;
      body: string;
      imageUrl?: string;
      data?: Record<string, string>;
    },
  ) {
    return await this.notificationsService.sendNotificationToTopic(
      topic,
      {
        title: body.title,
        body: body.body,
        imageUrl: body.imageUrl,
      },
      body.data,
    );
  }
}
```

## Best Practices

### 1. Token Management

**⚠️ CRÍTICO**: Los tokens FCM pueden expirar o volverse inválidos. Es fundamental implementar limpieza periódica de tokens inactivos para mantener la base de datos optimizada y evitar errores de envío.

```typescript
// notifications/notifications.service.ts
async cleanupInactiveTokens() {
  // Remove tokens that haven't been updated in 30 days
  const thirtyDaysAgo = new Date();
  thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);

  await this.userTokenRepository
    .createQueryBuilder()
    .update(UserToken)
    .set({ isActive: false })
    .where('updatedAt < :date', { date: thirtyDaysAgo })
    .andWhere('isActive = :active', { active: true })
    .execute();
}
```

### 2. Error Handling

```typescript
// notifications/notifications.service.ts
async sendNotificationWithRetry(tokens: string[], notification: any, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await this.firebaseService.sendMulticastNotification(tokens, notification);
    } catch (error) {
      if (attempt === maxRetries) {
        throw error;
      }
      
      // Wait before retrying (exponential backoff)
      await new Promise(resolve => setTimeout(resolve, Math.pow(2, attempt) * 1000));
    }
  }
}
```

### 3. Rate Limiting

```typescript
// notifications/notifications.service.ts
import { RateLimiter } from 'limiter';

private rateLimiter = new RateLimiter({
  tokensPerInterval: 100,
  interval: 'minute',
});

async sendNotificationWithRateLimit(tokens: string[], notification: any) {
  await this.rateLimiter.removeTokens(1);
  return await this.firebaseService.sendMulticastNotification(tokens, notification);
}
```

### 4. Notification Templates

```typescript
// notifications/notification-templates.service.ts
@Injectable()
export class NotificationTemplatesService {
  async sendWelcomeNotification(userId: string, userName: string) {
    return await this.notificationsService.sendNotificationToUser(userId, {
      title: 'Welcome!',
      body: `Hi ${userName}, welcome to our app!`,
      imageUrl: 'https://your-app.com/welcome-image.jpg',
    }, {
      type: 'welcome',
      userId,
    });
  }

  async sendOrderConfirmation(userId: string, orderId: string) {
    return await this.notificationsService.sendNotificationToUser(userId, {
      title: 'Order Confirmed!',
      body: `Your order #${orderId} has been confirmed and is being processed.`,
      imageUrl: 'https://your-app.com/order-confirmed.jpg',
    }, {
      type: 'order_confirmation',
      orderId,
    });
  }
}
```

## Testing

### Unit Tests

```typescript
// notifications/notifications.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { NotificationsService } from './notifications.service';
import { FirebaseService } from '../firebase/firebase.service';

describe('NotificationsService', () => {
  let service: NotificationsService;
  let firebaseService: FirebaseService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        NotificationsService,
        {
          provide: FirebaseService,
          useValue: {
            sendMulticastNotification: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<NotificationsService>(NotificationsService);
    firebaseService = module.get<FirebaseService>(FirebaseService);
  });

  it('should send notification to user', async () => {
    const mockResponse = { successCount: 1, failureCount: 0 };
    jest.spyOn(firebaseService, 'sendMulticastNotification').mockResolvedValue(mockResponse);

    const result = await service.sendNotificationToUser('user-id', {
      title: 'Test',
      body: 'Test message',
    });

    expect(result).toEqual(mockResponse);
  });
});
```

## Useful Links

- [Firebase Admin SDK Documentation](https://firebase.google.com/docs/admin/setup)
- [Firebase Cloud Messaging API](https://firebase.google.com/docs/reference/admin/node/admin.messaging)
- [NestJS Documentation](https://docs.nestjs.com/)
- [Firebase Console](https://console.firebase.google.com/)

## Example Usage

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { FirebaseModule } from './firebase/firebase.module';
import { NotificationsModule } from './notifications/notifications.module';
import { UserToken } from './users/entities/user-token.entity';

@Module({
  imports: [
    TypeOrmModule.forFeature([UserToken]),
    FirebaseModule,
    NotificationsModule,
  ],
})
export class AppModule {}
```

```typescript
// Example usage in a service
@Injectable()
export class OrderService {
  constructor(private notificationsService: NotificationsService) {}

  async createOrder(userId: string, orderData: any) {
    // Create order logic...
    const order = await this.orderRepository.save(orderData);

    // Send notification
    await this.notificationsService.sendNotificationToUser(userId, {
      title: 'Order Created!',
      body: `Your order #${order.id} has been created successfully.`,
    }, {
      orderId: order.id,
      type: 'order_created',
    });

    return order;
  }
}
```

---

**Last updated: July 10, 2025**

Remember to keep your Firebase service account key secure and never commit it to version control. Use environment variables for all sensitive configuration. For production, consider implementing proper logging and monitoring for notification delivery rates. 