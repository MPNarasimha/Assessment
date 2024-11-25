# Assessment

# 1. First, create the Nest.js project and set up MongoDB integration with Mongoose.
nest new notification-preferences
cd notification-preferences
npm install @nestjs/mongoose mongoose class-validator class-transformer

# Add environment variables (e.g., MongoDB URI) to the .env file for MongoDB integration
MONGODB_URI=mongodb://localhost/notification-preferences

# 2. Data Models
# Define the two models: UserPreference and NotificationLog using Mongoose.
# UserPreference Model (user-preference.schema.ts)
import { Schema, Document } from 'mongoose';

export interface UserPreference extends Document {
  userId: string;
  email: string;
  preferences: {
    marketing: boolean;
    newsletter: boolean;
    updates: boolean;
    frequency: 'daily' | 'weekly' | 'monthly' | 'never';
    channels: {
      email: boolean;
      sms: boolean;
      push: boolean;
    };
  };
  timezone: string;
  lastUpdated: Date;
  createdAt: Date;
}

export const UserPreferenceSchema = new Schema<UserPreference>({
  userId: { type: String, required: true },
  email: { type: String, required: true },
  preferences: {
    marketing: { type: Boolean, required: true },
    newsletter: { type: Boolean, required: true },
    updates: { type: Boolean, required: true },
    frequency: { type: String, enum: ['daily', 'weekly', 'monthly', 'never'], required: true },
    channels: {
      email: { type: Boolean, required: true },
      sms: { type: Boolean, required: true },
      push: { type: Boolean, required: true }
    }
  },
  timezone: { type: String, required: true },
  lastUpdated: { type: Date, default: Date.now },
  createdAt: { type: Date, default: Date.now }
});

# NotificationLog Model (notification-log.schema.ts)
import { Schema, Document } from 'mongoose';

export interface NotificationLog extends Document {
  userId: string;
  type: 'marketing' | 'newsletter' | 'updates';
  channel: 'email' | 'sms' | 'push';
  status: 'pending' | 'sent' | 'failed';
  sentAt?: Date;
  failureReason?: string;
  metadata: Record<string, any>;
}

export const NotificationLogSchema = new Schema<NotificationLog>({
  userId: { type: String, required: true },
  type: { type: String, enum: ['marketing', 'newsletter', 'updates'], required: true },
  channel: { type: String, enum: ['email', 'sms', 'push'], required: true },
  status: { type: String, enum: ['pending', 'sent', 'failed'], required: true },
  sentAt: { type: Date },
  failureReason: { type: String },
  metadata: { type: Object, required: true }
});

# 3. Service Layer
# Create services to handle the business logic for managing user preferences and sending notifications.
# UserPreferencesService (user-preference.service.ts)
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { UserPreference } from './user-preference.schema';

@Injectable()
export class UserPreferencesService {
  constructor(@InjectModel('UserPreference') private readonly userPreferenceModel: Model<UserPreference>) {}

  async createPreference(createPreferenceDto: any): Promise<UserPreference> {
    const newPreference = new this.userPreferenceModel(createPreferenceDto);
    return await newPreference.save();
  }

  async getPreference(userId: string): Promise<UserPreference> {
    return await this.userPreferenceModel.findOne({ userId }).exec();
  }

  async updatePreference(userId: string, updateDto: any): Promise<UserPreference> {
    return await this.userPreferenceModel.findOneAndUpdate({ userId }, updateDto, { new: true }).exec();
  }

  async deletePreference(userId: string): Promise<void> {
    await this.userPreferenceModel.deleteOne({ userId }).exec();
  }
}

# NotificationService (notification.service.ts)
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { NotificationLog } from './notification-log.schema';

@Injectable()
export class NotificationService {
  constructor(@InjectModel('NotificationLog') private readonly notificationLogModel: Model<NotificationLog>) {}

  async sendNotification(userId: string, type: string, channel: string, content: any): Promise<NotificationLog> {
    const log = new this.notificationLogModel({
      userId,
      type,
      channel,
      status: 'pending',
      metadata: content
    });
    await log.save();
    // Simulate sending the notification
    log.status = 'sent';
    log.sentAt = new Date();
    return await log.save();
  }

  async getNotificationLogs(userId: string): Promise<NotificationLog[]> {
    return await this.notificationLogModel.find({ userId }).exec();
  }
}

# 4. Controllers
# Create controllers for handling HTTP requests and mapping them to the service methods.
# UserPreferencesController (user-preference.controller.ts)
import { Controller, Post, Get, Patch, Delete, Param, Body } from '@nestjs/common';
import { UserPreferencesService } from './user-preference.service';

@Controller('api/preferences')
export class UserPreferencesController {
  constructor(private readonly userPreferencesService: UserPreferencesService) {}

  @Post()
  async create(@Body() createPreferenceDto: any) {
    return await this.userPreferencesService.createPreference(createPreferenceDto);
  }

  @Get(':userId')
  async get(@Param('userId') userId: string) {
    return await this.userPreferencesService.getPreference(userId);
  }

  @Patch(':userId')
  async update(@Param('userId') userId: string, @Body() updateDto: any) {
    return await this.userPreferencesService.updatePreference(userId, updateDto);
  }

  @Delete(':userId')
  async delete(@Param('userId') userId: string) {
    await this.userPreferencesService.deletePreference(userId);
    return { message: 'Preference deleted successfully' };
  }
}

# NotificationController (notification.controller.ts)
import { Controller, Post, Get, Param, Body } from '@nestjs/common';
import { NotificationService } from './notification.service';

@Controller('api/notifications')
export class NotificationController {
  constructor(private readonly notificationService: NotificationService) {}

  @Post('send')
  async sendNotification(@Body() sendNotificationDto: any) {
    return await this.notificationService.sendNotification(
      sendNotificationDto.userId,
      sendNotificationDto.type,
      sendNotificationDto.channel,
      sendNotificationDto.content
    );
  }

  @Get(':userId/logs')
  async getNotificationLogs(@Param('userId') userId: string) {
    return await this.notificationService.getNotificationLogs(userId);
  }
}

# 5.Validation and Error Handling
# Use class-validator and class-transformer to handle validation of incoming data.
# Create DTOs for validating user preferences and notification requests.
# CreatePreferenceDto (create-preference.dto.ts)
import { IsString, IsBoolean, IsEnum, IsObject, IsEmail, IsNotEmpty } from 'class-validator';

export class CreatePreferenceDto {
  @IsString()
  @IsNotEmpty()
  userId: string;

  @IsEmail()
  email: string;

  @IsObject()
  preferences: {
    marketing: boolean;
    newsletter: boolean;
    updates: boolean;
    frequency: 'daily' | 'weekly' | 'monthly' | 'never';
    channels: {
      email: boolean;
      sms: boolean;
      push: boolean;
    };
  };

  @IsString()
  timezone: string;
}

Use similar DTOs for the notification send request and validation.

# 6. Testing
# Write unit tests for services and controllers, mocking external dependencies.
# Example test for 
UserPreferencesService
import { Test, TestingModule } from '@nestjs/testing';
import { UserPreferencesService } from './user-preference.service';
import { getModelToken } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { UserPreference } from './user-preference.schema';

describe('UserPreferencesService', () => {
  let service: UserPreferencesService;
  let model: Model<UserPreference>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserPreferencesService,
        {
          provide: getModelToken('UserPreference'),
          useValue: {
            new: jest.fn().mockResolvedValue(mockUserPreference),
            constructor: jest.fn().mockResolvedValue(mockUserPreference),
          },
        },
      ],
    }).compile();

    service = module.get<UserPreferencesService>(UserPreferencesService);
    model = module.get<Model<UserPreference>>(getModelToken('UserPreference'));
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });
});
