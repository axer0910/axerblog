---
title: Doctrine自动添加时间戳
tag: php
date: 2019-03-06
updated: 2019-03-06
---


这里要注意的是，php中有规范可以识别注释的信息，所以注释要按照Doctrine的文档规范来书写。mysql如果类型是timestamp，可以按照和datetime相同方法处理（插入日期使用new \DateTime()）


```php
<?php

namespace Tranz\BMTestBundle\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * TestPage
 *
 * @ORM\Table(name="test_page")
 * @ORM\Entity
 * @ORM\HasLifecycleCallbacks()   //这里引入doctrine生命周期管理的函数
 */
class TestPage
{
    /**
     * @var \DateTime
     *
     * @ORM\Column(name="created_time", type="datetime", nullable=false)
     */
    private $createdTime = 'CURRENT_TIMESTAMP';

    /**
     * @var \DateTime
     *
     * @ORM\Column(name="updated_time", type="datetime", nullable=false)
     */
    private $updatedTime = 'CURRENT_TIMESTAMP';

    /**
     * @return \DateTime
     */
    public function getCreatedTime()
    {
        return $this->createdTime;
    }

    /**
     * @param \DateTime $createdTime
     */
    public function setCreatedTime($createdTime)
    {
        $this->createdTime = $createdTime;
    }

    /**
     * @return \DateTime
     */
    public function getUpdatedTime()
    {
        return $this->updatedTime;
    }

    /**
     * @param \DateTime $updatedTime
     */
    public function setUpdatedTime($updatedTime)
    {
        $this->updatedTime = $updatedTime;
    }

    /**
     * @ORM\Prepersist   //每次在commit前都会执行这个函数，达到自动更新创建时间和更新时间
     */
    public function PrePersist(){
        if($this->getCreatedTime()==null){
            $this->setCreatedTime(new \DateTime('now'));
        }
        $this->setUpdatedTime(new \DateTime('now'));
    }
}
```