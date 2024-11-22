# user-notification-preferences-api-challenges
Below is the full implementation for the *User Notification Preferences API Challenge*:

---

## *Folder and File Structure*

src/
|-- modules/
|   |-- preferences/
|   |   |-- preferences.controller.ts
|   |   |-- preferences.service.ts
|   |   |-- preferences.schema.ts
|   |   |-- preferences.dto.ts
|   |-- notifications/
|       |-- notifications.controller.ts
|       |-- notifications.service.ts
|       |-- notifications.schema.ts
|       |-- notifications.dto.ts
|-- common/
|   |-- pipes/
|   |   |-- validation.pipe.ts
|   |-- filters/
|       |-- http-exception.filter.ts
|-- app.module.ts
|-- main.ts
|-- environment.ts
|-- test/
|   |-- unit/
|   |   |-- preferences.service.spec.ts
|   |   |-- notifications.service.spec.ts
|   |-- e2e/
|       |-- app.e2e-spec.ts


---

## *Code Implementation*

### *1. App Setup*

#### *main.ts*
typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
  await app.listen(process.env.PORT || 3000);
}
bootstrap();


#### *app.module.ts*
typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { PreferencesModule } from './modules/preferences/preferences.module';
import { NotificationsModule } from './modules/notifications/notifications.module';

@Module({
  imports: [
    MongooseModule.forRoot(process.env.MONGO_URI),
    PreferencesModule,
    NotificationsModule,
  ],
})
export class AppModule {}


---

### *2. Preferences Module*

#### *preferences.schema.ts*
typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

@Schema({ timestamps: true })
export class UserPreference extends Document {
  @Prop({ required: true })
  userId: string;

  @Prop({ required: true, unique: true })
  email: string;

  @Prop({
    type: Object,
    required: true,
    validate: {
      validator: (value) => typeof value === 'object',
      message: 'Invalid preferences object',
    },
  })
  preferences: {
    marketing: boolean;
    newsletter: boolean;
    updates: boolean;
    frequency: 'daily' | 'weekly' | 'monthly' | 'never';
    channels: { email: boolean; sms: boolean; push: boolean };
  };

  @Prop({ required: true })
  timezone: string;

  @Prop()
  lastUpdated: Date;
}

export const UserPreferenceSchema = SchemaFactory.createForClass(UserPreference);


#### *preferences.dto.ts*
typescript
import { IsEmail, IsEnum, IsObject, IsString, ValidateNested } from 'class-validator';
import { Type } from 'class-transformer';

class ChannelsDto {
  @IsString()
  email: boolean;

  @IsString()
  sms: boolean;

  @IsString()
  push: boolean;
}

class PreferencesDto {
  @IsString()
  marketing: boolean;

  @IsString()
  newsletter: boolean;

  @IsString()
  updates: boolean;

  @IsEnum(['daily', 'weekly', 'monthly', 'never'])
  frequency: string;

  @ValidateNested()
  @Type(() => ChannelsDto)
  channels: ChannelsDto;
}

export class CreatePreferenceDto {
  @IsString()
  userId: string;

  @IsEmail()
  email: string;

  @ValidateNested()
  @Type(() => PreferencesDto)
  preferences: PreferencesDto;

  @IsString()
  timezone: string;
}


#### *preferences.service.ts*
typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { UserPreference } from './preferences.schema';
import { CreatePreferenceDto } from './preferences.dto';

@Injectable()
export class PreferencesService {
  constructor(
    @InjectModel(UserPreference.name) private userModel: Model<UserPreference>,
  ) {}

  async create(createPreferenceDto: CreatePreferenceDto): Promise<UserPreference> {
    return new this.userModel(createPreferenceDto).save();
  }

  async findOne(userId: string): Promise<UserPreference> {
    const user = await this.userModel.findOne({ userId });
    if (!user) throw new NotFoundException('User not found');
    return user;
  }

  async update(userId: string, updateData: Partial<CreatePreferenceDto>) {
    const user = await this.userModel.findOneAndUpdate(
      { userId },
      updateData,
      { new: true },
    );
    if (!user) throw new NotFoundException('User not found');
    return user;
  }

  async delete(userId: string): Promise<void> {
    await this.userModel.findOneAndDelete({ userId });
  }
}


#### *preferences.controller.ts*
typescript
import { Controller, Get, Post, Patch, Delete, Param, Body } from '@nestjs/common';
import { PreferencesService } from './preferences.service';
import { CreatePreferenceDto } from './preferences.dto';

@Controller('preferences')
export class PreferencesController {
  constructor(private preferencesService: PreferencesService) {}

  @Post()
  create(@Body() createPreferenceDto: CreatePreferenceDto) {
    return this.preferencesService.create(createPreferenceDto);
  }

  @Get(':userId')
  findOne(@Param('userId') userId: string) {
    return this.preferencesService.findOne(userId);
  }

  @Patch(':userId')
  update(@Param('userId') userId: string, @Body() updateData: Partial<CreatePreferenceDto>) {
    return this.preferencesService.update(userId, updateData);
  }

  @Delete(':userId')
  delete(@Param('userId') userId: string) {
    return this.preferencesService.delete(userId);
  }
}


---

### *3. Notifications Module*

#### *notifications.schema.ts*
typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

@Schema({ timestamps: true })
export class NotificationLog extends Document {
  @Prop({ required: true })
  userId: string;

  @Prop({ required: true })
  type: 'marketing' | 'newsletter' | 'updates';

  @Prop({ required: true })
  channel: 'email' | 'sms' | 'push';

  @Prop({ required: true })
  status: 'pending' | 'sent' | 'failed';

  @Prop()
  sentAt?: Date;

  @Prop()
  failureReason?: string;

  @Prop({ type: Object })
  metadata: Record<string, any>;
}

export const NotificationLogSchema = SchemaFactory.createForClass(NotificationLog);


#### *notifications.controller.ts*
typescript
import { Controller, Post, Body } from '@nestjs/common';
import { NotificationsService } from './notifications.service';

@Controller('notifications')
export class NotificationsController {
  constructor(private notificationsService: NotificationsService) {}

  @Post('/send')
  send(@Body() notificationData: any) {
    return this.notificationsService.send(notificationData);
  }
}


---

### *Tests*

- Use Jest for unit tests for services.
- Write E2E tests for APIs using supertest.

---

### *Deployment*

1. Deploy using [Vercel](https://vercel.com) or [AWS Lambda](https://aws.amazon.com/lambda/).
2. Add environment variables in .env.

---
